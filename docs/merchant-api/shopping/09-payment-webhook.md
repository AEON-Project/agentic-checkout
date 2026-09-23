# Payment Webhook

**Brief Description**

- For orders where `webhookUrl` was passed at payment time, when the order reaches a terminal state (`COMPLETED` payment succeeded / `FAIL` failed and closed /
  `TIMEOUT` payment authorization timed out), the server POSTs the payment result as JSON to that URL; the merchant must receive, process, and respond according to
  the response specification below
- The same notification may be delivered multiple times (retries, network replays); the merchant must handle it idempotently: first check by `outTradeNo` (or
  `orderNo`) whether the business data has already been processed, and if so respond success directly; acquire a data lock for concurrency control before the
  status check and processing, to avoid re-entry
- The merchant must verify the signature (recompute and compare with its own `secret` per the "Signature" rules), and verify that the `paidUsd` amount matches the merchant-side
  order amount, to prevent financial loss from "fake notifications"
- The signature scope is the flat fields: the two object fields `pricing` / `receipt` do not participate in the signature; amount verification is based on the top-level
  `paidUsd` (which participates in the signature); other fields whose value is null or `""` do not participate in the signature (same rules as "Signature")

**Webhook Method**

- POST `webhookUrl`, `Content-Type: application/json`, timeout 5 seconds
- The request body is JSON; the common parameters `appId` / `userId` / `sign` are at the top level (alongside the business fields);
  see "Signature" for the signature rules

**Webhook Parameters**

| Parameter | Type | Description |
|---|---|---|
| appId | string | Merchant identifier (participates in the signature) |
| userId | string | User email (participates in the signature) |
| notifyType | string | Fixed `ORDER_RESULT` (terminal-state result notification; reserved for extension) |
| notifyId | string | Unique ID of this notification. Unchanged when the same notification is resent; the idempotency deduplication key |
| notifyTime | string | Send time of this delivery (ISO-8601 UTC; updated on resend) |
| orderNo | string | System order number |
| outTradeNo | string | Merchant external order number (the value passed at payment time is echoed back as-is; the merchant uses it to match its own order) |
| channelOrderId | string | Channel order ID |
| channel | string | `SHOPIFY` / `AMAZON` / `TRAVALA` |
| title | string | Product/booking title |
| status | string | Terminal state: `COMPLETED` / `FAIL` / `TIMEOUT` (same semantics as the status table in "Payment Detail") |
| failCode | string | Failure code, non-null only when `FAIL`; values are the same as "Error Codes" |
| reason | string | End-user-facing failure reason in English, non-null only on failure |
| payNo | string | Payment order number (charge idempotency key, reconciliation evidence); null for failed orders aborted before charging |
| paidUsd | string | Actual charged amount (USD). Null if not charged (including `TIMEOUT` and failures before charging) — amount verification is based on this field (participates in the signature) |
| pricing | object | Real cost breakdown and actual charge, same structure as `pricing` in "Payment Detail"; null for failed orders whose pricing was never confirmed. Does not participate in the signature |
| receipt | object | Success receipt, same structure as `receipt` in "Payment Detail", non-null only when `COMPLETED`. Does not participate in the signature |
| sign | string | SHA-512 signature (computed over the flat fields above), does not participate in the signature |

**Webhook Example**

```json
{
  "appId": "TEST000001",
  "userId": "user@example.com",
  "notifyType": "ORDER_RESULT",
  "notifyId": "NTF17889441230011a2b3c4d",
  "notifyTime": "2026-09-09T17:12:42Z",
  "orderNo": "AIS17889...",
  "outTradeNo": "M20260915170001",
  "channelOrderId": "hWNGc...",
  "channel": "SHOPIFY",
  "title": "4-in-1 USB-C Hub — Space Gray",
  "status": "COMPLETED",
  "failCode": null,
  "reason": null,
  "payNo": "AIP17889...",
  "paidUsd": "26.09",
  "pricing": { "subtotal": "19.99", "shipping": "4.90", "tax": "1.20", "fee": null,
               "total": "26.09", "currency": "USD", "fxRate": "1",
               "paidUsd": "26.09", "payMethod": "CARD", "payToken": "USDT" },
  "receipt": { "merchantOrderNumber": "#1042", "statusUrl": "https://acmegear.com/.../orders/...",
               "items": ["4-in-1 USB-C Hub — Space Gray ×1"],
               "paidAt": "2026-09-09T17:12:30Z", "paymentRecordNo": "WR17889..." },
  "sign": "<SHA-512 uppercase>"
}
```

**Response Specification**

- Merchant processed successfully: HTTP status code **200** and the response body contains the string **`success`** (e.g. plain text `success`,
  or `{"result":"success"}`); if either condition is missing, this notification is judged failed and enters retry
- Signature verification failed / amount mismatch: do not return `success` (respond as a failure), and raise an alert on the merchant side for investigation

**Retry Policy**

- After the first notification fails, resends follow a decaying frequency: every 2 minutes within 10 minutes after the order reaches a terminal state, every 10 minutes within 1 hour,
  every 1 hour within 12 hours; stops after 12 hours (about 20 attempts in total)
- During retries, the merchant may call "Payment Detail" at any time to get the authoritative status without waiting for notifications
- An invalid `webhookUrl` (not https / unreachable / certificate error) does not affect the payment flow; only the notification fails
