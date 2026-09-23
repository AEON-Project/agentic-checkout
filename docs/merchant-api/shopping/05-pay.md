# Pay

**Brief Description**

- Before calling this endpoint, you must obtain the user's explicit confirmation of the product, variant, and amount
- Idempotency is keyed by the merchant's order number: `outTradeNo` is the payment order number in the merchant's own system (required). Repeated pay calls with the same `outTradeNo` will not cause duplicate charges — the existing `orderNo` and current status are returned directly.


**Request Method**

- POST `/orders/pay`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for the generation rules |
| channelOrderId | Yes | string | Returned in the Create Order response |
| channel | Yes | string | Channel |
| payMethod | Yes | string | `CARD` card payment (SHOPIFY / AMAZON) / `WALLET` wallet payment (TRAVALA only) |
| token | No | string | Required only for `payMethod=WALLET` (TRAVALA): symbol of the wallet asset to debit (e.g. `USDT`), taken from "Asset Balance"; its `usdValue` must be enough to cover the booking total — missing returns `91017`, insufficient value returns `91007` (both synchronously at acceptance: no charge, no order created). Do not pass for `CARD` (SHOPIFY / AMAZON) |
| outTradeNo | Yes | string | Merchant external order number (the payment order number in the merchant's own system; must be unique within the same merchant). Idempotency key |
| webhookUrl | No | string | Order webhook URL (must start with `https://` and be reachable from the public internet). The server proactively notifies this URL when the order reaches a terminal state; see "Order Webhook" for the specification. If not passed, no notification is sent |

**Parameter Examples** (complete examples per channel)

SHOPIFY (card payment, with webhook URL):

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "channelOrderId": "hWNGc...", "channel": "SHOPIFY", "payMethod": "CARD",
  "outTradeNo": "M20260915170001",
  "webhookUrl": "https://merchant.example.com/aeon/order-notify" }
```

AMAZON (card payment):

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "channelOrderId": "amz_cb88902a23b24ffd9cb68101faa87c19", "channel": "AMAZON", "payMethod": "CARD",
  "outTradeNo": "M20260915170002" }
```

TRAVALA (wallet payment, `token` required):

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "channelOrderId": "c58f2db4a7e94b0f9d3126aa84c1c77e", "channel": "TRAVALA", "payMethod": "WALLET",
  "token": "USDT",
  "outTradeNo": "M20260915170003",
  "webhookUrl": "https://merchant.example.com/aeon/order-notify" }
```

**Response Parameters**

| Parameter | Type | Description |
|---|---|---|
| orderNo | string | System order number; always use it for order tracking |
| channelOrderId | string | Channel order ID echoed back |
| status | string | Status after acceptance (usually `PROCESSING`) |

**Response Examples** (identical structure across channels; only the order-number shapes differ)

SHOPIFY:

```json
{ "orderNo": "AIS1788944123001101", "channelOrderId": "hWNGc...", "status": "PROCESSING" }
```

AMAZON:

```json
{ "orderNo": "AIS1788944156002102", "channelOrderId": "amz_cb88902a23b24ffd9cb68101faa87c19", "status": "PROCESSING" }
```

TRAVALA:

```json
{ "orderNo": "AIS1788944189003103", "channelOrderId": "c58f2db4a7e94b0f9d3126aa84c1c77e", "status": "PROCESSING" }
```

##### Payment Rules

- Card payment (SHOPIFY / AMAZON): the charge only uses the balance already on the AI Card and **will never automatically transfer from the wallet to top up the card**.
  Before paying, you must complete "Initialize AI Card" and use "Transfer Wallet to AI Card" to stock up the card balance. Balance verification happens in two stages:
  (1) **Synchronous pre-check at acceptance** — if the card is not activated, `91009` is returned synchronously; if the card balance is not even enough to cover the quoted `payTotal`,
  `91007` is returned synchronously, **no order is created** and no funds are moved — call "Transfer Wallet to AI Card" to top up, then retry
  (the `outTradeNo` has not been consumed; you may reuse it or use a new one); if the card balance query fails at acceptance, `91021` is returned synchronously — likewise
  no order is created; retry later. The same applies to TRAVALA wallet payment: if the value of the passed `token` is not enough to cover the quote, `91007` is returned synchronously at acceptance
  and no funds are moved;
  (2) **Final check against the real checkout total** — the real total (SHOPIFY includes shipping and tax) may be higher than the quote. If the card balance covers the quote
  but not the real total, the order fails asynchronously: `failCode=91007`, no funds are moved, and `reason` carries the card balance and the payable total
  figures; top up the card balance, then create a new order and pay with a new `outTradeNo`. If the **card balance query fails** at this stage (card-issuing channel
  outage), `failCode=91021` is returned and no funds are moved — this is not an insufficient balance, **no top-up is needed**; simply retry later with a new `outTradeNo`;
- If the real total significantly exceeds the quote, the order is aborted before charging (`91006`), with no charge made;
- AMAZON: `quote.total` is the authorization ceiling (product price + sales-tax buffer). The card is charged the **actually charged amount** at Amazon checkout
  (≤ the ceiling). 3DS verification during checkout is completed automatically by the server; the merchant does not need to handle `pendingAction`.
  **Direct payment with a payment method stored in the Amazon account**: when the user's Amazon account already has a payment method bound (the user's own card, or a card
  retained from a previous order), the order may be completed directly with that payment method — in this case **the AI Card is not charged**, and the cost is collected by Amazon
  directly from the user's bound payment method. In both paths, `paidUsd` in the order/webhook is the amount actually paid by the user
  (≤ `quote.total`), `pricing.payToken` is null, and the funds come from the user in either case; merchant-side handling logic
  is identical, and amount reconciliation should always use `paidUsd`;
- Spending limits (per transaction / per day, per user) apply; exceeding a limit returns `91011`;
- Failures that can be determined at acceptance are returned synchronously as errors (no order is created);
