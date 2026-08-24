**English** | [简体中文](energy-api.zh-CN.md)

# Energy API

> Online version: https://www.trxapi.io/docs/energy.html · API version v1

The TRXAPI Energy API is a small set of REST endpoints for integrating TRON energy delegation into your own systems: exchange withdrawal pipelines, payment-merchant payouts, bulk-transfer tools. Core flow: **order (getenergy) → poll or wait for the webhook (order / webhook) → reconcile (balance)**. Every order returns an on-chain transaction hash `txid` you can verify independently in any block explorer.

- Billing: charged per order from your **prepaid balance** (topped up in the customer center); failed orders are auto-refunded
- Energy specs: `65000` / `130000`; durations: `1` / `24` hours

## Conventions

- Base URL `https://api.trxapi.io`, auth via `?apikey=YOUR_API_KEY`
- Unified response envelope (**branch on `code`, never on the message text**):

```jsonc
{
  "code": 200,          // 200 = success, otherwise an error code
  "message": "SUCCESS",
  "data": { /* endpoint-specific payload */ },
  "traceId": "7c2c1d6f..."
}
```

- TRON addresses are Base58 (`T`-prefixed, 34 chars); amount fields are strings denominated in TRX
- All business endpoints are `GET` + query string; the URL contains `&`, so always quote it in a shell

## Endpoint overview

| Endpoint | Method | Description |
|---|---|---|
| `/api/v1/openapi/getenergy` | GET | Rent energy (65000/130000, 1h/24h) |
| `/api/v1/openapi/activation` | GET | Activate a never-used TRON address |
| `/api/v1/openapi/order` | GET | Query order status and txid by orderId |
| `/api/v1/openapi/balance` | GET | Query the merchant prepaid balance |
| `/api/v1/openapi/node/…` | GET/POST | Direct node queries, free (see [Node API](node-api.md)) |
| Webhook | POST (server → you) | Push notification on terminal order states, configured in the customer center |

## Rent energy

`GET /api/v1/openapi/getenergy`

| Param | Required | Description |
|---|---|---|
| `apikey` | Yes | Merchant key, created in the customer center |
| `add` | Yes | TRON address to receive energy |
| `value` | Yes | Energy amount — only 65000 or 130000 |
| `hour` | Yes | Duration in hours — only 1 or 24 |
| `traceId` | No | Trace ID (≤36 chars), echoed back for reconciliation |

Response `data`: `orderId`, `amount` (fee in TRX), `balance` (after the charge), `txid` (on-chain delegation hash).

```bash
curl "https://api.trxapi.io/api/v1/openapi/getenergy?apikey=YOUR_API_KEY&add=TR7NHqjeKQx...&value=65000&hour=1"
```

> ⚠️ The receiving address must already be activated (at least one on-chain transaction) — call the activation endpoint first otherwise.

## Address activation

`GET /api/v1/openapi/activation` — params `apikey`, `add`. Returns `add` / `txid` / `amount`. Already-activated addresses return `702001` (not billed; safe to repeat).

## Query order

`GET /api/v1/openapi/order` — params `apikey`, `orderId`. Status: `processing` / `completed` / `failed` (auto-refunded). Poll every 2–3 seconds until terminal, or use the webhook.

## Query balance

`GET /api/v1/openapi/balance` — param `apikey`. Returns `balance` (prepaid, TRX-denominated) and `currency`. This is your TRXAPI service-account balance, unrelated to any external wallet.

## Order Callback (Webhook)

When an order reaches a terminal state, the server POSTs a notification to the callback URL you configured in the customer center. The webhook is **account-level**: all API keys share one URL and signing secret.

**Body**: `event` (order.completed / order.failed), `orderId`, `status`, `txid` (empty on failure), `timestamp` (Unix seconds).

**Headers**: `X-Timestamp` (send time), `X-Signature` (`sha256=<hex(HMAC-SHA256)>`).

**Verification** (always sign over the RAW body):

```text
sig = "sha256=" + hex(hmac_sha256(secret, ts + "." + rawBody))
```

```python
# Python
import hmac, hashlib

def verify(secret: str, ts: str, raw_body: bytes, header_sig: str) -> bool:
    mac = hmac.new(secret.encode(), f"{ts}.".encode() + raw_body, hashlib.sha256)
    return hmac.compare_digest("sha256=" + mac.hexdigest(), header_sig)
```

```js
// Node.js (use the raw body, never re-serialized JSON)
const crypto = require('crypto')
function verify(secret, ts, rawBody, headerSig) {
  const expected = 'sha256=' + crypto.createHmac('sha256', secret).update(ts + '.' + rawBody).digest('hex')
  return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(headerSig))
}
```

**Retries**: HTTP 2xx acknowledges; otherwise backoff at 1m / 5m / 30m / 2h / 6h, up to 6 attempts. Check the timestamp drift (e.g. within 5 minutes) against replays, and handle idempotently by orderId.

## Error codes

| code | Description |
|---|---|
| **200** | Success — see data |
| 500 | Bad params / order failed / temporarily unavailable — retry later |
| 501 | apikey missing or disabled |
| 502 | Insufficient prepaid balance — top up |
| 429 | Rate limited (direct node API only) |
| 702001 | Address already activated (not billed) |
| 100002 | Delegation channel busy — retry later (not billed; deductions auto-refunded) |

## Best practices

- **traceId reconciliation**: pass your system's unique business ID; it is echoed back
- **Guard against duplicate orders**: a GET resent blindly after a timeout can double-charge — lock on your business ID and query the order before re-ordering
- **Retry with restraint**: wait a few seconds on 100002; check params first on 500
- **Validate addresses before ordering**: format check + run new addresses through activation
- **Balance alerts**: poll the balance endpoint and alert below a threshold to avoid 502 at peak
- **Key hygiene**: apikey server-side only, injected via environment variables; webhooks must be verified + idempotent

## Code example (order → poll)

```python
import time, requests

BASE = "https://api.trxapi.io/api/v1/openapi"
KEY  = "YOUR_API_KEY"

def rent(address, value=65000, hour=1, trace=None):
    params = {"apikey": KEY, "add": address, "value": value, "hour": hour}
    if trace: params["traceId"] = trace
    r = requests.get(f"{BASE}/getenergy", params=params, timeout=15).json()
    assert r["code"] == 200, r
    return r["data"]["orderId"]

def wait_done(order_id, timeout=60):
    deadline = time.time() + timeout
    while time.time() < deadline:
        r = requests.get(f"{BASE}/order", params={"apikey": KEY, "orderId": order_id}, timeout=15).json()
        if r["code"] == 200 and r["data"]["status"] in ("completed", "failed"):
            return r["data"]
        time.sleep(2)
    raise TimeoutError(order_id)

order_id = rent("TR7NHqjeKQx...", trace="withdraw-8801")
print(wait_done(order_id))
```

Machine-readable spec: [openapi-energy.yaml](openapi-energy.yaml). Support: admin@coinepay.cc
