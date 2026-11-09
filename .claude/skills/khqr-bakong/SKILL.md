---
name: khqr-bakong
description: Expert guide for accepting payments with Cambodia's Bakong KHQR (the national EMVCo QR standard every Cambodian banking app can scan). Covers how a KHQR string is built (TLV tags, CRC-16, MD5), generating dynamic QR for an order, checking payment through the Bakong Open API (check_transaction_by_md5), polling/reconciliation, deep links for mobile, and the security rules for marking an order paid. Framework- and language-agnostic (Next.js, Express, Flask/Django, Laravel, Go, …). Use when the user mentions KHQR, Bakong, NBC, Cambodian QR payment, ABA/ACLEDA/Wing QR, "bakong-khqr" packages, or wants checkout/payments for a business in Cambodia.
---

# KHQR / Bakong payments

You are acting as someone who has shipped Bakong KHQR checkouts before. Use this file for the mental model and the rules; open the reference files only for the part you're working on:

| Need | Read |
|---|---|
| Build or decode the QR string, tag table, CRC, worked example | `references/khqr-format.md` |
| Bakong Open API: token, endpoints, responses, errors, limits | `references/bakong-api.md` |
| Checkout architecture, order state machine, schema, polling, reconciliation, security checklist | `references/payment-flow.md` |
| Drop-in encoder code (TypeScript, Python, PHP, Go) and library comparison | `references/encoders.md` |

## Mental model (read this first)

1. **A KHQR is just a string.** An EMVCo merchant-presented QR payload: `tag(2) + length(2) + value` fields, ending with a CRC-16 (`6304XXXX`). You can build it locally with no network call. The QR image is only that string rendered as a QR code.
2. **The MD5 of the exact string is the payment's ID.** `md5(qrString)` (lowercase hex) is what you later ask Bakong about. Store the string *and* its MD5 at creation time; never regenerate, because the string contains a creation timestamp and will hash differently.
3. **Bakong has no webhook.** You find out about a payment only by asking: `POST https://api-bakong.nbc.gov.kh/v1/check_transaction_by_md5` with a Bearer developer token. `responseCode: 0` + `data` means paid. So every integration needs polling while the customer waits plus a background reconciliation for customers who close the tab.
4. **Money goes straight to a Bakong account** (`name@bank`, e.g. `shop@aclb`, `shop@abaa`). Nobody sits in between: no refunds API, no disputes, no card payments. Refunds are a manual bank transfer.
5. **Dynamic vs static.** Checkout always uses *dynamic* QR (`010212`, has amount + expiration). Static QR (`010211`, no amount) is a printed sticker; never use it for orders, because you can't tie a payment to an order.

## Non-negotiable rules

- **Server builds everything.** Amount, currency, account ID and bill number come from the server's own order record, never from the request body (the classic tutorial bug is `<input type="hidden" name="amount">`).
- **"I have paid" proves nothing.** A button may *trigger* a check; only a Bakong API response may mark an order paid.
- **Verify the response before trusting it:** `data.toAccountId` equals your configured account, `data.amount` equals the order total, `data.currency` matches. Store `data.hash` with a UNIQUE constraint so one bank transaction can never settle two orders.
- **Idempotent settlement.** The poller, the "I've paid" button and the reconciliation job can all fire at once. Transition `pending → paid` with a conditional update (`WHERE status = 'pending'`) and only run side effects (stock, email, bag clear) for the request that won.
- **Token is a server secret.** Never ship it to the browser, never log it. It expires (reported ~90 days); plan a renewal reminder (`/v1/renew_token`).
- **Expire QR codes** (10–15 min for web checkout) and put the expiry in tag 99. Treat a payment that arrives after expiry as paid-but-flagged, not as lost money (see payment-flow.md → late payments).
- **Keep merchant name/city ASCII** in tags 59/60 (≤25 / ≤15 chars). Khmer text belongs in tag 64 (alternate language). Length is counted in characters; non-ASCII there breaks length/CRC agreement in some apps.
- **Third-party relays** (e.g. `rbk_…` tokens that route to `api.khqr.dev` in the Python `bakong-khqr` lib) send your transactions through someone else's server. Use the NBC endpoint directly unless the user consciously picks a relay.

## Workflow when asked to integrate KHQR

1. **Find what exists:** framework, DB/ORM, how orders/cart are modeled, money representation (cents? floats?), deployment host/region, auth or guest checkout.
2. **Confirm inputs with the user** if missing: Bakong account ID (`x@bank`), merchant name/city, currency (USD/KHR), whether they already have a Bakong developer token (register at the Bakong Open API portal, https://api-bakong.nbc.gov.kh). If they're on a bank gateway (ABA PayWay etc.) instead, say so — that's a different API.
3. **Check reachability early:** the Bakong API is reported to reject non-Cambodian IPs (HTTP 403). If the server runs abroad (Vercel/AWS outside KH), test a call from the deploy target *before* building the flow; fall back to a KH-hosted proxy/VPS if blocked.
4. **Model the order first** (payment-flow.md → schema), then the encoder (encoders.md), then the Bakong client (bakong-api.md), then the status endpoint + polling UI, then reconciliation.
5. **Test** with a real small payment (minimum 100 KHR or USD 0.01) from a banking app: scan, pay, watch status flip, confirm idempotency by hitting the check endpoint twice, let one QR expire.

## Choosing a library vs your own encoder

The format is small and stable; a ~60-line encoder you own is often better than a dependency. Summary (details in encoders.md):

- **JS/TS:** official NBC `bakong-khqr` (npm, CommonJS, no types, old axios dep; its README shows the wrong `IndividualInfo` argument order — real order is `(bakongAccountID, merchantName, merchantCity, optional)`), or community `ts-khqr` (typed), or own encoder.
- **Python:** `bakong-khqr` (bsthen) — `create_qr`, `generate_md5`, `check_payment`, `get_payment`, `check_bulk_payments`, `generate_deeplink`. Note `expiration` there is in **days**.
- **Anything else:** write the encoder from khqr-format.md and call the HTTP API directly.

Whatever you pick, validate output with a CRC check and by scanning with a real banking app before shipping.
