# Bakong Open API

Base URL: `https://api-bakong.nbc.gov.kh/v1`
All calls: `POST`, `Content-Type: application/json`, `Authorization: Bearer <developer token>` (except token endpoints).

Facts marked *(reported)* come from client libraries / community docs, not an official NBC statement. Re-check them against the current portal when it matters.

## Getting a token

1. Register with an email on the Bakong Open API portal (`https://api-bakong.nbc.gov.kh`, "Register"). A token is emailed / shown.
2. Token is a JWT that expires — *(reported)* ~90 days. Renew with `POST /v1/renew_token` `{ "email": "<registered email>" }` → `{ responseCode: 0, responseMessage: "Token has been issued", data: { token } }`.
3. Store as a server-only env var (`BAKONG_TOKEN`). Add a monitor/reminder: an expired token makes every check fail (looks like "nobody is paying").

## Response envelope

```json
{ "responseCode": 0, "responseMessage": "...", "errorCode": null, "data": { ... } }
```
- `responseCode: 0` → success; `1` → failure, see `errorCode`.
- HTTP errors: `401` bad/expired token; `403` *(reported)* request not from a Cambodian IP; `429` rate limited; `5xx`/`504` Bakong busy — retry with backoff.

## Endpoints

### check_transaction_by_md5 — the one you need
Request: `{ "md5": "<32-char lowercase md5 of the QR string>" }`

Paid → `responseCode: 0` with `data`:
| Field | Meaning |
|---|---|
| `hash` | Bakong transaction hash — store, UNIQUE |
| `fromAccountId` | payer's Bakong account |
| `toAccountId` | receiver — **must equal your account** |
| `currency` | `USD` / `KHR` — must match the order |
| `amount` | number — must equal the order total |
| `description` | free text |
| `createdDateMs`, `acknowledgedDateMs` | Unix ms |
| `externalRef`, `instructionRef`, `trackingStatus`, `receiverBank`, `receiverBankAccount` | may be present; don't depend on them |

Not paid → `responseCode: 1`, `errorCode: 1` (*transaction not found*: not paid yet, or wrong md5). `errorCode: 3` → *(reported)* transaction failed. Treat both as "not paid", keep polling until the QR expires.

### check_transaction_by_md5_list — reconciliation
Request: JSON **array** of md5 strings, max 50. Response `data` is an array of `{ md5, status, ... }`; `status: "SUCCESS"` means paid. Use for a cron that sweeps pending orders. Still call the single endpoint (or use the returned item) to get the full `data` for verification.

### check_transaction_by_hash / check_transaction_by_short_hash
Look up by the bank transaction hash (full or 8-char short hash + amount + currency). Useful when a customer sends a receipt screenshot and the md5 path failed.

### generate_deeplink_by_qr — mobile UX
```json
{ "qr": "<khqr string>",
  "sourceInfo": { "appIconUrl": "https://…/icon.png", "appName": "Atelier", "appDeepLinkCallback": "https://shop.example/checkout/…" } }
```
→ `data.shortLink` — open it on a phone to jump into a banking app with the payment prefilled. All three `sourceInfo` fields are required if `sourceInfo` is sent.

### check_bakong_account
`{ "accountId": "name@bank" }` → `errorCode 11` = doesn't exist, `12` = invalid format. Use once at setup to validate config, not per checkout.

## Polling strategy

- While the customer is on the payment screen: every **3–5 s**, stop at expiry or on paid.
- The Python `bakong-khqr` lib's suggested backoff: 5 s for the first 5 min, 10 s to 15 min, 15 s to 1 h, then 5 min.
- Poll **your own server**, which checks its DB first and only calls Bakong if still pending (keeps the token server-side, collapses duplicate checks, respects rate limits).
- Reconciliation job every 1–5 min: batch pending, unexpired-or-recently-expired orders through `check_transaction_by_md5_list`.

## Timeouts & errors

- Client timeout ~10–15 s. Timeout ≠ unpaid: leave the order pending and check again later.
- Never mark an order failed because of a 5xx/timeout; only expiry ends a pending order.
- Log `responseCode`, `errorCode`, HTTP status, md5 — never the token.

## Hosting caveat

*(reported)* The API returns 403 for non-Cambodian source IPs. Serverless hosts (Vercel, Netlify, AWS Lambda) run outside Cambodia. Test from the real deploy target on day one. Options if blocked: a small KH-hosted VPS/proxy that holds the token and exposes only "check md5" to your app (authenticated), or a bank gateway (ABA PayWay) instead.
