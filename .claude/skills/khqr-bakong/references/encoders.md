# KHQR encoders

The TypeScript and Python encoders were run on 2026-11-05: both reproduce the official SDK sample CRC (`58D8`) and produce byte-identical QR + MD5 for the same input. The PHP and Go snippets follow the same algorithm but haven't been run; run the self-test at the bottom before relying on them. They build a **dynamic individual (tag 29)** QR. For a merchant QR (tag 30), swap the template tag and add sub 01 merchant ID, 02 acquiring bank.

Inputs: `accountId`, `merchantName`, `merchantCity`, `currency` (`USD`|`KHR`), `amount` (string already formatted: `"12.50"` or `"15000"`), `billNumber`, optional `storeLabel`, `terminalLabel`, `mobileNumber`, `expiresAtMs`, `nowMs`.

Validate lengths before encoding (see khqr-format.md) and throw on overflow; never truncate silently.

## TypeScript / JavaScript (Node, Bun, Deno, edge)

```ts
import { createHash } from "node:crypto"; // edge runtimes: use a small md5 lib instead

const tlv = (tag: string, value: string) => {
  if (value.length > 99) throw new Error(`KHQR tag ${tag} too long`);
  return tag + String(value.length).padStart(2, "0") + value;
};

export function crc16(payload: string): string {
  let crc = 0xffff;
  for (const byte of new TextEncoder().encode(payload)) {
    crc ^= byte << 8;
    for (let i = 0; i < 8; i++) crc = crc & 0x8000 ? ((crc << 1) ^ 0x1021) & 0xffff : (crc << 1) & 0xffff;
  }
  return crc.toString(16).toUpperCase().padStart(4, "0");
}

export type KhqrInput = {
  accountId: string; merchantName: string; merchantCity: string;
  currency: "USD" | "KHR"; amount: string; billNumber: string;
  storeLabel?: string; terminalLabel?: string; mobileNumber?: string;
  expiresAtMs: number; nowMs?: number; merchantCategoryCode?: string;
};

export function buildKhqr(i: KhqrInput): { qr: string; md5: string } {
  const max = (v: string | undefined, n: number, f: string) => {
    if (v && v.length > n) throw new Error(`${f} exceeds ${n} characters`);
  };
  max(i.accountId, 32, "accountId"); max(i.merchantName, 25, "merchantName");
  max(i.merchantCity, 15, "merchantCity"); max(i.amount, 13, "amount");
  max(i.billNumber, 25, "billNumber"); max(i.storeLabel, 25, "storeLabel");
  max(i.terminalLabel, 25, "terminalLabel"); max(i.mobileNumber, 25, "mobileNumber");

  const now = i.nowMs ?? Date.now();
  if (i.expiresAtMs <= now) throw new Error("expiresAtMs must be in the future");

  const additional =
    (i.billNumber ? tlv("01", i.billNumber) : "") +
    (i.mobileNumber ? tlv("02", i.mobileNumber) : "") +
    (i.storeLabel ? tlv("03", i.storeLabel) : "") +
    (i.terminalLabel ? tlv("07", i.terminalLabel) : "");

  let payload =
    tlv("00", "01") +
    tlv("01", "12") +
    tlv("29", tlv("00", i.accountId)) +
    tlv("52", i.merchantCategoryCode ?? "5999") +
    tlv("53", i.currency === "USD" ? "840" : "116") +
    tlv("54", i.amount) +
    tlv("58", "KH") +
    tlv("59", i.merchantName) +
    tlv("60", i.merchantCity) +
    (additional ? tlv("62", additional) : "") +
    tlv("99", tlv("00", String(now)) + tlv("01", String(i.expiresAtMs))) +
    "6304";
  payload += crc16(payload);
  return { qr: payload, md5: createHash("md5").update(payload, "utf8").digest("hex") };
}

export const formatAmount = (minor: number, currency: "USD" | "KHR") =>
  currency === "USD" ? (minor / 100).toFixed(2) : String(minor);
```

Bakong check (Node 18+ `fetch`):

```ts
export async function checkByMd5(md5: string) {
  const res = await fetch("https://api-bakong.nbc.gov.kh/v1/check_transaction_by_md5", {
    method: "POST",
    headers: { "Content-Type": "application/json", Authorization: `Bearer ${process.env.BAKONG_TOKEN}` },
    body: JSON.stringify({ md5 }),
    cache: "no-store",
    signal: AbortSignal.timeout(12_000),
  });
  if (!res.ok) throw new Error(`Bakong HTTP ${res.status}`); // 401 token, 403 IP, 429 rate, 5xx busy
  return (await res.json()) as {
    responseCode: number; errorCode: number | null; responseMessage: string;
    data: { hash: string; fromAccountId: string; toAccountId: string; currency: string;
            amount: number; description?: string; createdDateMs: number; acknowledgedDateMs: number } | null;
  };
}
```

## Python

```python
import hashlib, time

def tlv(tag: str, value: str) -> str:
    return f"{tag}{len(value):02d}{value}"

def crc16(payload: str) -> str:
    crc = 0xFFFF
    for b in payload.encode("utf-8"):
        crc ^= b << 8
        for _ in range(8):
            crc = ((crc << 1) ^ 0x1021) & 0xFFFF if crc & 0x8000 else (crc << 1) & 0xFFFF
    return f"{crc:04X}"

def build_khqr(account_id, merchant_name, merchant_city, currency, amount: str, bill_number,
               expires_at_ms: int, store_label=None, terminal_label=None, now_ms=None):
    now = now_ms or int(time.time() * 1000)
    add = (tlv("01", bill_number) if bill_number else "") \
        + (tlv("03", store_label) if store_label else "") \
        + (tlv("07", terminal_label) if terminal_label else "")
    p = (tlv("00", "01") + tlv("01", "12") + tlv("29", tlv("00", account_id)) + tlv("52", "5999")
         + tlv("53", "840" if currency == "USD" else "116") + tlv("54", amount) + tlv("58", "KH")
         + tlv("59", merchant_name) + tlv("60", merchant_city)
         + (tlv("62", add) if add else "")
         + tlv("99", tlv("00", str(now)) + tlv("01", str(expires_at_ms))) + "6304")
    p += crc16(p)
    return p, hashlib.md5(p.encode("utf-8")).hexdigest()
```

Or the library: `pip install bakong-khqr` → `KHQR(token).create_qr(account_id=…, merchant_name=…, merchant_city=…, amount=…, currency="USD", bill_number=…, static=False, expiration=1)` (expiration in **days**), `generate_md5(qr)`, `check_payment(md5)` → `"PAID"|"UNPAID"`, `get_payment(md5)` → data dict. Beware: tokens starting with `rbk` silently switch it to the third-party `api.khqr.dev` relay.

## PHP

```php
function tlv(string $t, string $v): string { return $t . str_pad((string)strlen($v), 2, '0', STR_PAD_LEFT) . $v; }
function crc16(string $p): string {
    $crc = 0xFFFF;
    foreach (unpack('C*', $p) as $b) {
        $crc ^= $b << 8;
        for ($i = 0; $i < 8; $i++) $crc = ($crc & 0x8000) ? (($crc << 1) ^ 0x1021) & 0xFFFF : ($crc << 1) & 0xFFFF;
    }
    return strtoupper(str_pad(dechex($crc), 4, '0', STR_PAD_LEFT));
}
// build like the TS version; md5($qr) for the lookup key.
// strlen counts bytes: fine for ASCII fields; use mb_strlen for tag 64 Khmer text and test with a real app.
```

## Go

```go
func tlv(tag, v string) string { return fmt.Sprintf("%s%02d%s", tag, len(v), v) }
func crc16(p string) string {
	crc := uint16(0xFFFF)
	for _, b := range []byte(p) {
		crc ^= uint16(b) << 8
		for i := 0; i < 8; i++ {
			if crc&0x8000 != 0 { crc = crc<<1 ^ 0x1021 } else { crc <<= 1 }
		}
	}
	return fmt.Sprintf("%04X", crc)
}
// md5: hex.EncodeToString(md5.Sum([]byte(qr))[:])
```

## Self-test

Rebuild the SDK sample and expect CRC `58D8`:
tag 30 (not 29) with `0015john_smith@devb`, MCC 5999, USD, amount `500.0`, name `John Smith`, city `Phnom Penh`, additional `01 Invoice#069`, `03 Coffee Khlaing`, `07 Cooooooooooooooooooounter 2`, timestamps created `1727340564907`, expires `1727340624907`.

## Library comparison

| | Official `bakong-khqr` (npm) | `ts-khqr` (npm) | `bakong-khqr` (PyPI) | Own encoder |
|---|---|---|---|---|
| Maintainer | NBC | community | community (bsthen) | you |
| Types | none (add a `.d.ts`) | yes | n/a | yes |
| Includes API client | deeplink, account check (axios 0.24) | partial | yes (check, bulk, deeplink) | write ~20 lines with fetch |
| Gotchas | README `IndividualInfo` arg order is wrong; returns `{status:{code:0 ok / 1 err}, data:{qr,md5}}` instead of throwing; expiration in **ms** | check version/maintenance | expiration in **days**; `rbk` tokens go to relay | must test with real apps |
