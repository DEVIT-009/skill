# KHQR string format

KHQR follows EMVCo Merchant-Presented QR (MPM). Each field is **TLV**: 2-digit tag, 2-digit zero-padded length (characters), value. Some values are themselves TLV lists ("templates"). Fields are emitted in ascending tag order, except CRC (63) which is always last.

```
tlv(tag, value) = tag + pad2(len(value)) + value
```

## Top-level tags

| Tag | Name | Value | Notes |
|---|---|---|---|
| 00 | Payload format indicator | `01` | always `000201` |
| 01 | Point of initiation | `11` static / `12` dynamic | dynamic = has amount + expiry; use for checkout |
| 15 | UnionPay merchant account | string ≤99 | optional, KHR only |
| 29 | Merchant account info — **individual** | template, see below | most shops/personal accounts |
| 30 | Merchant account info — **merchant** | template, see below | registered merchants with merchant ID + acquiring bank |
| 52 | Merchant category code | 4 digits, default `5999` | ISO 18245 MCC |
| 53 | Transaction currency | `840` USD / `116` KHR | ISO 4217 numeric |
| 54 | Transaction amount | ≤13 chars | dynamic only; USD max 2 decimals (`12.50`), KHR integer (`15000`) |
| 58 | Country code | `KH` | |
| 59 | Merchant name | ≤25 chars | keep ASCII |
| 60 | Merchant city | ≤15 chars | keep ASCII, e.g. `Phnom Penh` |
| 62 | Additional data | template, see below | omit entirely if all subfields empty |
| 64 | Merchant info — alternate language | template | Khmer name/city go here |
| 99 | Timestamp | template | creation + expiration (ms) |
| 63 | CRC | 4 hex uppercase | always last |

### Tag 29 (individual) subfields
| Sub | Name | Max |
|---|---|---|
| 00 | Bakong account ID (`name@bank`) | 32 |
| 01 | Account information (e.g. bank account no.) | 32 |
| 02 | Acquiring bank | 32 |

### Tag 30 (merchant) subfields
| Sub | Name | Max |
|---|---|---|
| 00 | Bakong account ID | 32 |
| 01 | Merchant ID | 32 |
| 02 | Acquiring bank | 32 |

### Tag 62 (additional data) subfields
| Sub | Name | Max | Use |
|---|---|---|---|
| 01 | Bill number | 25 | **your order reference** — make it unique per order |
| 02 | Mobile number | 25 | digits; local format `0…` (strip `855`) |
| 03 | Store label | 25 | shop/branch name |
| 07 | Terminal label | 25 | e.g. `Web`, `POS-01` |
| 08 | Purpose of transaction | 25 | |

### Tag 64 (language template)
| Sub | Name | Max |
|---|---|---|
| 00 | Language preference | 2 (e.g. `km`) — required if 64 present |
| 01 | Merchant name, alternate language | 25 |
| 02 | Merchant city, alternate language | 15 |

### Tag 99 (timestamp)
| Sub | Name | Value |
|---|---|---|
| 00 | Creation timestamp | Unix **milliseconds**, 13 digits |
| 01 | Expiration timestamp | Unix ms, 13 digits; **required for dynamic QR** by the official SDK; must be in the future and after creation |

## CRC

CRC-16/CCITT-FALSE: polynomial `0x1021`, init `0xFFFF`, no reflection, no final XOR, over the **UTF-8 bytes** of everything *including the literal `6304`*. Output 4 uppercase hex digits appended after `6304`.

```
payload = fields + "6304"
qr      = payload + hex4(crc16(payload))
```

Verification: `crc16(qr[:-4]) == qr[-4:].upper()`.

## MD5

`md5 = lowercase_hex(md5(utf8(qr)))` of the **complete** string including CRC. This is the lookup key for the Bakong API. Store it; never recompute from a regenerated QR (timestamps change).

## Worked example (verified)

Official SDK sample (CRC `58D8` verified):

```
00020101021230190015john_smith@devb5204599953038405405500.05802KH5910John Smith6010Phnom Penh62640111Invoice#0690314Coffee Khlaing0727Cooooooooooooooooooounter 299340013172734056490701131727340624907630458D8
```

Decoded:
```
00 "01"
01 "12"                      dynamic
30 "0015john_smith@devb"     merchant template: 00=john_smith@devb
52 "5999"
53 "840"                     USD
54 "500.0"
58 "KH"
59 "John Smith"
60 "Phnom Penh"
62 01="Invoice#069" 03="Coffee Khlaing" 07="Cooooooooooooooooooounter 2"
99 00="1727340564907" 01="1727340624907"   (created, expires +60s)
63 "58D8"
```

Built for an order ($12.50 USD, individual account):
```
000201 010212 2916 0012atelier@aclb 52045999 5303840 540512.50 5802KH 5913Atelier Store 6010Phnom Penh 6230 0108ORD-0001 0307Atelier 0703Web 9934 0013<created> 0113<expires> 6304<CRC>
```
(spaces added for reading only)

## Decoding

Walk the string: read 2-char tag, 2-char length, slice value, advance `4 + length`. Recurse into 29, 30, 62, 64, 99. Reject if CRC fails, if tags are out of order, or if required tags (00, 29|30, 52, 53, 58, 59, 60, 63) are missing.

## Rendering

- Render the string with any QR library (error correction M is fine). Server-side SVG or data URL avoids shipping the string-building logic to the client.
- NBC publishes KHQR brand guidelines (red header with KHQR logo, merchant name, amount, currency). Following them makes customers trust the code; it is optional for function.
- On mobile, customers can't scan their own screen — offer a **deep link** (bakong-api.md → `generate_deeplink_by_qr`) and/or "save QR image" so they can open it from the banking app's gallery.
