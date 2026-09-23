# Integration Overview

Enables a merchant system to search for products and complete purchases on behalf of its users. Funds come from the user's own AEON wallet, payment is executed server-side, and the merchant never touches card data at any point. The merchant must ensure it has obtained the user's authorization for purchase-on-behalf / pay-on-behalf operations.

- Base URL: `https://<host>/agentapi/v1`
- All endpoints uniformly use **POST + JSON** (UTF-8); all timestamps are ISO-8601 UTC
- Common parameters `appId` / `userId` / `sign` go at the top level of the request body JSON; every request is signed (see "Signature")
- The HTTP status code is always 200; success or failure is always determined by the envelope `code` (see "Error Codes")
- Channels: `SHOPIFY` / `AMAZON` / `TRAVALA`; catalog and order request/response structures differ by channel

##### Unified Response

The "response examples" in each endpoint's documentation show only the `model` content:

```json
{
  "code": "0",
  "msg": "success",
  "model": { ... },
  "traceId": "6745a17bddef1b1497bade4f13f3f531",
  "success": true,
  "error": false
}
```

| Field | Description |
|---|---|
| `code` | Numeric code as a string; `"0"` means success, non-zero is an error code |
| `msg` | Result message, English by default, safe to display directly to the user |
| `model` | Business data (null when there is no data) |
| `traceId` | Trace ID; include it when reporting issues |
| `success` | Boolean derived from `code`; true on success, mutually exclusive with `error` |
| `error` | Boolean derived from `code`; true on failure, mutually exclusive with `success` |

##### Call Sequence

The call order for a complete purchase:

```
⓪ Initialize account   POST /account/init (must be called first for every new userId on first integration; only opens the wallet;
                any other endpoint called with an uninitialized appId+userId always returns 91028)
⓪ Initialize AI Card  POST /wallet/card/init (paid card creation: deducts tokens from the wallet, minimum value 0.6 USD;
                SHOPIFY / AMAZON channels pay by card; must be completed before payment)
                → POST /wallet/card/status  poll cardStatus until ACTIVE
① Shipping address     POST /address/get (if missing, save one via POST /address/save)
② Search        POST /search
③ Detail        POST /items/detail          pick a variantId / asin / packageId
④ Check balance     POST /wallet/assets
⑤ Create order (lock the quote)     POST /orders/create         → channelOrderId + quote
        ── Recap the product and amount to the user and obtain explicit consent ──
⑥ Pay        POST /orders/pay            → returns orderNo (you must pass your merchant payment number outTradeNo as the idempotency key;
                also recommended to pass webhookUrl to receive final-state webhooks)
⑦ Poll        POST /orders/detail every 5–10 seconds
                ├─ pendingAction non-null → ask the user for the verification code → POST /orders/actions
                ├─ COMPLETED           → report the merchant order number
                ├─ FAIL                → handle per failCode, display reason verbatim
                └─ PROCESSING          → keep polling
⑧ Webhook        When the order reaches a final state (COMPLETED / FAIL / TIMEOUT) the server POSTs to webhookUrl
                (verify the signature + process idempotently by outTradeNo; on failure retries with a decaying schedule for about 12 hours, see "Payment Webhook")
```

Polling and webhooks complement each other: webhooks free the merchant from long polling and deliver the final state promptly, but notifications may be delayed or lost.
**The authoritative status is always "Payment Detail"**; after receiving a webhook it is recommended to query once more before posting to your books.

##### Order Number Glossary

- `channelOrderId`: the channel order ID returned by create order (locks the quote, does not move funds)
- `outTradeNo`: the merchant's own payment number passed in at pay time (idempotency key; resubmitting with the same number returns the existing order)
- `orderNo`: the system order number returned by pay; always use it for order tracking
- `payNo`: the payment number, used as the reconciliation reference

The payment method is determined by the channel: SHOPIFY / AMAZON pay by card (`CARD`, deducting the AI Card's existing balance; funds are not automatically transferred from the wallet to top up the card), TRAVALA pays by wallet (`WALLET`).
Duplicate payments are blocked idempotently (resubmitting with the same `outTradeNo` or the same `channelOrderId` both return the existing order, never charging twice), and definite payment failures are automatically refunded via the original route; the merchant side does not need to implement any fund-protection logic.

##### Limits

- Spending limits (per user): per-transaction and daily USD caps; exceeding them fails before any funds are touched (`91011`).
- API rate limits (per user); exceeding them returns `91014`, retry after backoff:

| Tier | Quota | Covered endpoints |
|---|---|---|
| Catalog | 30 req/min | `/search`, `/items/detail` |
| Funds | 5 req/min | `/orders/create`, `/orders/pay`, `/orders/actions`, `/orders/booking/cancel`, `/wallet/card/init`, `/wallet/card/topup`, `/wallet/card/redeem`, `/wallet/token/redeem` |
| Query | No rate limit | All other endpoints (order query / assets / address etc. are read-only, backed by caches / lightweight reads) |

- Duplicate protection: repeated pay calls with the same `outTradeNo` or the same `channelOrderId` are idempotent and return the existing order, never charging twice. The server does no product-level duplicate protection; whether the same product can be purchased again is managed by the merchant via its own `outTradeNo`.

##### Integrator Rules

1. The `secret` must be stored only on the merchant's servers; it must never be leaked or shipped to apps / web pages / end devices. If leaked, contact operations immediately to rotate it. Do not write `sign` or request parameters into publicly accessible logs.
2. The merchant must ensure it has obtained the user's authorization for purchase-on-behalf / pay-on-behalf operations; before payment, recap the product, variant, and quoted total to the user (for `ESTIMATE` channels, note that shipping and taxes may be added), and only call `pay` after obtaining explicit consent.
3. Display the failure `reason` to the end user verbatim; during `PROCESSING`, do not promise refunds on the server's behalf or fabricate outcomes.
4. Poll every 5–10 seconds; on `91014` (rate limit), use exponential backoff.
5. Versioning and compatibility: breaking changes go to a new version path (`/agentapi/v2`); adding response fields or error codes within v1 is not a breaking change — deserialization must ignore unknown fields, and unknown error codes should be handled as a generic failure.
