# Create Order (locks the quote, no funds movement)

**Brief Description**

- Creates an order on the channel side and locks the quote, returning the channel order ID (`channelOrderId`) and the quote; no funds are moved
- Each order is fixed at 1 item; to buy multiple items, create multiple orders
- If the create-order request times out or fails over the network, it is safe to retry; unpaid channel orders expire and are voided automatically

**Request Method**

- POST `/orders/create`

**Parameters** (use the fields corresponding to `channel`)

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for the generation rules |
| channel | Yes | string | `SHOPIFY` / `AMAZON` / `TRAVALA` |
| variantId | SHOPIFY: Yes | string | The variant to purchase (`variants[].variantId` from the detail response) |
| shopDomain | SHOPIFY: Yes | string | The `merchantDomain` of that variant |
| country | SHOPIFY: Yes | string | Same as used in search/detail; for physical goods it must match the country of the saved address |
| asin | AMAZON: Yes | string | From the detail response. AMAZON only supports US shipping addresses |
| packageId | TRAVALA: Yes | string | The room package selected by the user |
| sessionId | TRAVALA: Yes | string | Returned by the hotel search within the same session |
| guest.firstName | TRAVALA: Yes | string | Guest first name |
| guest.lastName | TRAVALA: Yes | string | Guest last name, used for identity verification in subsequent order queries/cancellations |
| guest.phone | TRAVALA: Yes | string | Guest phone number, including the international dialing code (e.g. `+8529319534025`) |

**Parameter Examples**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "channel": "SHOPIFY",
  "variantId": "gid://shopify/ProductVariant/4409...",
  "shopDomain": "acmegear.com",
  "country": "US" }
```

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "channel": "AMAZON", "asin": "B0DXJQT19B" }
```

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "channel": "TRAVALA",
  "packageId": "PKG_abc123",
  "sessionId": "sess_9f2c...",
  "guest": { "firstName": "Waters", "lastName": "Alexander", "phone": "+8529319534025" } }
```

**Response Parameters**

| Parameter | Type | Description |
|---|---|---|
| channel | string | Channel echoed back |
| channelOrderId | string | Channel order ID; pass it to "Pay" |
| quote.title | string | Product/booking title |
| quote.pricingType | string | `ESTIMATE` (SHOPIFY: shipping and tax are confirmed at payment time) or `FINAL` (TRAVALA: the actual charged amount; AMAZON: the authorized maximum — the actual charge is ≤ this amount, with `paidUsd` in Order Detail as the source of truth) |
| quote.currency | string | Pricing currency (ISO 4217); all amount fields below are in this currency |
| quote.subtotal | string | Item subtotal (returned if known at this moment; null if not applicable/unknown; see the pricing matrix in "Order Detail" for its composition) |
| quote.shipping | string | Shipping (returned if known at this moment; null if not applicable/unknown; see the pricing matrix in "Order Detail" for its composition) |
| quote.tax | string | Tax (returned if known at this moment; null if not applicable/unknown; see the pricing matrix in "Order Detail" for its composition) |
| quote.fee | string | Fee (returned if known at this moment; null if not applicable/unknown; see the pricing matrix in "Order Detail" for its composition) |
| quote.total | string | Current total payable: `FINAL` = actual charged amount; `ESTIMATE` = locked quote amount (excluding subsequent shipping and tax) |
| quote.fxRate | string | Exchange rate of `currency` against USD (`"1"` when `currency=USD`) |
| quote.payTotalUsd | string | Payable amount converted to USD; use this when restating the amount to the user |
| quote.note | string | English note, can be shown as-is; null if absent |
| quoteExpiresAt | string | Quote validity: SHOPIFY / AMAZON about 30 minutes, TRAVALA about 5 minutes. After expiry, payment returns `91005`; simply create the order again |

**Response Examples** (same structure across channels)

SHOPIFY:

```json
{
  "channel": "SHOPIFY",
  "channelOrderId": "hWNGc...",
  "quote": {
    "title": "4-in-1 USB-C Hub — Space Gray",
    "pricingType": "ESTIMATE",
    "currency": "EUR",
    "subtotal": "18.50", "shipping": null, "tax": null, "fee": null,
    "total": "18.50",
    "fxRate": "1.083",
    "payTotalUsd": "20.04",
    "note": "Final total may include shipping/tax confirmed at checkout."
  },
  "quoteExpiresAt": "2026-09-09T17:30:00Z"
}
```

AMAZON:

```json
{
  "channel": "AMAZON",
  "channelOrderId": "amz_cb88902a23b24ffd9cb68101faa87c19",
  "quote": {
    "title": "Anker USB C Hub, 7in1 Multi-Port USB Adapter, 4K@60Hz USBC to HDMI Splitter",
    "pricingType": "FINAL",
    "currency": "USD",
    "subtotal": null,
    "shipping": null,
    "tax": null,
    "fee": "0.00",
    "total": "24.19",
    "fxRate": "1",
    "payTotalUsd": "24.19",
    "note": "Total is the authorized maximum (item price + sales-tax buffer). The card is charged the actual checkout amount, which may be lower."
  },
  "quoteExpiresAt": "2026-09-17T10:31:20Z"
}
```

TRAVALA (`title` = hotel name — room type name; `subtotal` = room price, `fee` = fee, both in USD):

```json
{
  "channel": "TRAVALA",
  "channelOrderId": "c58f2db4a7e94b0f9d3126aa84c1c77e",
  "quote": {
    "title": "Ramada by Wyndham Manila Central — Classic Room, 2 Twin Beds, City View",
    "pricingType": "FINAL",
    "currency": "USD",
    "subtotal": "83.88",
    "shipping": null,
    "tax": null,
    "fee": "0.00",
    "total": "83.88",
    "fxRate": "1",
    "payTotalUsd": "83.88",
    "note": null
  },
  "quoteExpiresAt": "2026-09-17T10:42:00Z"
}
```
