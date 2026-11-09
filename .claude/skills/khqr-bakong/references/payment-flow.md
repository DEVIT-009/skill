# KHQR checkout flow (framework-agnostic)

## Sequence

```
Customer          Browser                    Your server                         Bakong API
   │  Place order    │                            │                                   │
   │────────────────▶│ POST /checkout (form)      │                                   │
   │                 │───────────────────────────▶│ 1. re-price cart from DB          │
   │                 │                            │ 2. reserve stock + insert order   │
   │                 │                            │    (status=pending, expires_at)   │
   │                 │                            │ 3. build KHQR, md5 → save on order│
   │                 │◀───────── redirect /pay/{order} (QR + amount + countdown)      │
   │  scan & pay in  │                            │                                   │
   │  banking app ──────────────────────────────────────────────────────────────────▶│
   │                 │ poll GET /orders/{id}/status every 3–5s                        │
   │                 │───────────────────────────▶│ pending? → check_transaction_by_md5
   │                 │                            │──────────────────────────────────▶│
   │                 │                            │◀── responseCode 0 + data ─────────│
   │                 │                            │ 4. verify to/amount/currency      │
   │                 │                            │ 5. pending→paid (conditional),    │
   │                 │                            │    store hash (UNIQUE), side fx   │
   │                 │◀── { status: "paid" } ─────│                                   │
   │                 │ → /thank-you?order=…       │                                   │
                                  cron every 1–5 min: sweep pending via md5_list; expire stale
```

## Variant: check on click only

Some owners want Bakong asked **only when the customer clicks "I've paid"**: no polling, no cron. This is valid, and simpler (no status route, no scheduler, one API call per click). Build it like this:
- The QR is shown in a modal with the check button directly under it; the button calls a server action that runs the settlement routine once.
- Expire overdue orders lazily (e.g. at the start of the next order creation) with a DB-only statement. No Bakong call.
- When a click finds a payment for an `expired` order, mark it `paid_late`.
- Tell the owner the trade-off: a customer who pays and never clicks leaves the order unsettled although the money arrived. Offer reconciliation as an opt-in later.

## Order state machine

```
pending ──paid (verified)──▶ paid ──▶ (fulfilled / shipped … your domain)
   │
   ├─expires_at passed, still unpaid──▶ expired   (release reserved stock)
   └─customer cancels──────────────────▶ cancelled (release reserved stock)

expired/cancelled + payment found later ──▶ paid_late  (needs human: fulfil if stock, else refund by bank transfer)
```

Only one transition path writes `paid`: a conditional update `UPDATE orders SET status='paid', … WHERE id=$1 AND status='pending'` that returns the row. Zero rows → someone else already settled it (or it expired) → read and return current status, run **no** side effects.

## Minimal schema

```
orders
  id               uuid pk
  public_ref       text unique        -- bill number in tag 62/01, e.g. "AT7K2M9QX" (≤25 chars)
  user_id          fk nullable        -- guest checkout allowed?
  status           enum pending|paid|expired|cancelled|paid_late
  currency         'USD' | 'KHR'
  subtotal / shipping / total (integer minor units, CHECK >= 0)
  contact + shipping address columns
  khqr_string      text not null      -- exact string shown to customer
  khqr_md5         char(32) unique not null
  expires_at       timestamptz not null
  paid_at          timestamptz
  bakong_hash      text unique        -- prevents one payment settling two orders
  payer_account    text
  paid_amount      integer            -- what Bakong reported, minor units
  created_at / updated_at

order_items
  id, order_id fk (cascade or restrict), product_id fk restrict,
  product_name (snapshot), unit_price (snapshot, minor units), quantity CHECK > 0
```

Snapshots matter: the product price can change after the order; the order must keep what the customer was charged.

## Money conversion

Store integers. Convert only at the edges:
- USD cents → tag 54: `(cents / 100).toFixed(2)` → `"12.50"`. Bakong returns `amount: 12.5` → compare `Math.round(amount * 100) === totalCents`.
- KHR: whole riel, tag 54 `"15000"`; compare `amount === totalRiel`.
- Minimums: USD 0.01, KHR 100.

## Stock

Two options; pick per business:
- **Reserve at order creation** (recommended for low-stock/luxury): decrement in the same atomic write that inserts the order; release on expiry/cancel. Rely on a DB `CHECK (stock >= 0)` so oversell aborts the whole write.
- **Decrement at payment**: simpler, but two customers can pay for the last item. Then you need the `paid_late`/refund path more often.

## Endpoints (names are illustrative)

| Endpoint | Does |
|---|---|
| `POST /checkout` (form action / server action) | validate form, re-price, reserve, insert order + items, build QR, redirect to pay page |
| `GET /pay/{orderId}` (page) | show QR image, amount, merchant, countdown, deep-link button on mobile, "I've paid" button; authorize viewer |
| `GET /api/orders/{id}/status` | authorize; if `pending` and not expired → call Bakong; settle if paid; return `{status}` |
| `POST /api/orders/{id}/check` | same as status, for the "I've paid" button (just forces an immediate check) |
| cron `/api/cron/reconcile` | secret-protected; sweep pending (batch md5 list ≤50); expire overdue; release stock |

Authorize every order endpoint: the order belongs to the session user, or the browser holds a signed/httpOnly cookie with the order id for guests. An unguessable UUID alone leaks addresses if shared.

## Settlement routine (pseudocode)

```
settle(order):
  if order.status != pending: return order.status
  res = bakong.checkByMd5(order.khqr_md5)          # timeout → return pending
  if res.responseCode != 0: 
      if now > order.expires_at: expire(order)       # conditional pending→expired + release stock
      return current status
  d = res.data
  assert d.toAccountId == CONFIG.accountId
  assert d.currency == order.currency
  assert toMinor(d.amount, d.currency) == order.total
  if any assert fails: flag order for review, alert, return pending   # never auto-pay
  won = UPDATE orders SET status='paid', paid_at=…, bakong_hash=d.hash, payer_account=d.fromAccountId, paid_amount=…
        WHERE id=order.id AND status='pending' RETURNING id
  if won: run side effects once (clear cart, send email, revalidate caches)
  return 'paid'
```

Late payment: when the reconciler finds an `expired` order paid, set `paid_late` (conditional on `status='expired'`), alert staff. Do not silently re-reserve stock.

## Security checklist

- [ ] Amount/currency/account come from server config + DB, never the client
- [ ] Token only in server env; not in client bundles, logs, or error messages
- [ ] Bakong response verified: `toAccountId`, `amount`, `currency`
- [ ] `bakong_hash` UNIQUE, `khqr_md5` UNIQUE, `public_ref` UNIQUE
- [ ] Conditional `pending → paid` transition; side effects only for the winner
- [ ] Order endpoints authorize the viewer
- [ ] Cron endpoint requires a secret header
- [ ] Rate-limit the status/check endpoints per order (each call may hit Bakong)
- [ ] QR has an expiration timestamp; server enforces `expires_at` too
- [ ] Pay page and status responses are `no-store` (never cached/CDN'd)

## Framework recipes

- **Next.js (App Router):** checkout = Server Action (`"use server"`) that `redirect()`s to `/checkout/pay/[orderId]`; status = Route Handler `app/api/orders/[id]/status/route.ts` (dynamic, no cache); client component polls with `setInterval`/`fetch` and `router.replace()` on paid; QR via `qrcode` package `toString(qr, {type:"svg"})` in a server component; mark Bakong client file `import "server-only"`; cron via Vercel Cron → route handler with `CRON_SECRET`.
- **Express / Fastify:** same endpoints; render QR server-side; `node-cron` or platform scheduler.
- **Flask / Django:** server-render the pay page; small `fetch` poll script; Celery beat / APScheduler / cron for reconciliation. Python `bakong-khqr` lib covers QR + API.
- **Laravel:** controller + Blade/Livewire polling (`wire:poll.4s`); scheduler (`schedule:run`) for reconciliation; queue job for side effects.
- **Mobile app backends:** same server; app shows the deep link button prominently instead of the QR.
