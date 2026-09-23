# Order Detail

**Brief Description**

- Query payment detail information

**Request Method**

- POST `/orders/detail`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for the generation rules |
| orderNo | Either one | string | System order number returned by the pay response |
| outTradeNo | Either one | string | Merchant external order number passed at payment time (isolated per merchant; only orders of the current merchant can be queried) |

**Parameter Examples**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "orderNo": "AIS17889..." }
```

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "outTradeNo": "M20260915170001" }
```

**Response Parameters**

| Parameter | Type | Description |
|---|---|---|
| orderNo | string | Platform order number |
| outTradeNo | string | Merchant external order number (the value passed at payment time is echoed back as-is; the merchant uses it to match its own order) |
| channel | string | Channel |
| channelOrderId | string | Channel order ID |
| merchantDomain | string | Merchant domain (SHOPIFY) |
| title | string | Product/booking title (same as `quote.title` from Create Order) |
| status | string | Order status; see the status table below for the enum |
| failCode | string | Failure code, non-null only when `status=FAIL`; values are the same set as "Error Codes" |
| reason | string | End-user-facing failure reason in English |
| addressIssue | object | Structured details of an address problem, non-null only when `status=FAIL` and the failure is related to the shipping address: `error` = the checkout's exact error text (English, may be null), `validRegions` = the list of selectable states/provinces on the checkout form (may be null). Use it to programmatically fix "Save Address" and then create a new order |
| payNo | string | Latest payment order number (reconciliation evidence); null if no payment was made |
| pricing | object | Confirmed real cost breakdown and actual charge; see the table below for fields |
| pendingAction | object | Pending user action, non-null when user input is required (`status` remains `PROCESSING` at this point); see "Manual Verification" for handling |
| receipt | object | Success receipt, non-null only when `COMPLETED`; see the table below for fields |
| createdAt | string | Order creation time |
| updatedAt | string | Latest status change time |

`pricing` fields (all amounts are strings; line items the channel does not have are null):

| Parameter | Type | Description |
|---|---|---|
| subtotal | string | Product subtotal; see the pricing matrix below for composition |
| shipping | string | Shipping cost; see the pricing matrix below for composition |
| tax | string | Tax; see the pricing matrix below for composition |
| fee | string | Service fee; see the pricing matrix below for composition |
| total | string | Real payable total = sum of the non-null line items |
| currency | string | Pricing currency (ISO 4217) |
| fxRate | string | `currency` to USD exchange rate (`"1"` when `currency=USD`) |
| paidUsd | string | Actual charged amount (USD), = `total` × `fxRate` |
| payMethod | string | Payment method of this order: `CARD` / `WALLET` |
| payToken | string | Asset actually debited on the wallet side; null if the card balance was sufficient and the wallet was not touched |

Channel pricing matrix (Create Order `quote` and order detail `pricing` share the same structure):

| Channel | currency | Line-item composition | fxRate |
|---|---|---|---|
| SHOPIFY | Merchant fiat currency | `subtotal + shipping + tax` (`fee` is always null) | Fiat-to-USD exchange rate |
| AMAZON | USD | Create Order quote `total` = authorization ceiling (product price + sales-tax buffer); after the order succeeds, `subtotal`/`total`/`paidUsd` are fixed to the **actually charged amount** (≤ the ceiling; tax is merged into subtotal and `tax` is `"0.00"`), and `shipping`/`fee` are null. Card payment (debits the AI Card balance; when the user's Amazon account already has a bound payment method, Amazon collects directly and the card is not charged), `payToken` is always null | `"1"` |
| TRAVALA | USD | `subtotal` (room rate) `+ fee` (service fee, may be `"0"`) | `"1"` |

`receipt` fields (items the channel cannot provide are null):

| Parameter | Type | Description |
|---|---|---|
| merchantOrderNumber | string | Merchant-side order credential: SHOPIFY = merchant confirmation number (e.g. `#1042`); AMAZON = checkout order number (anchor for status query/reconciliation); TRAVALA = booking number (used together with the guest's `lastName` to manage/cancel the booking) |
| statusUrl | string | Merchant order status page link (no login required; check shipping/status afterwards). Only SHOPIFY may have it; null if the merchant does not provide one |
| items | array | Purchased content lines (for SHOPIFY these are the raw text lines scraped from the checkout confirmation page; the format is not guaranteed and they are for display/manual verification only) |
| shipTo | object | Shipping/guest snapshot (the actual delivery information at payment time); see the table below for fields |
| checkIn | string | Check-in date (TRAVALA only, `YYYY-MM-DD`) |
| checkOut | string | Check-out date (TRAVALA only, `YYYY-MM-DD`) |
| txHash | string | On-chain payout transaction hash (legacy field, always null for all channels currently) |
| paidAt | string | Payment completion time |
| paymentRecordNo | string | Platform charge record number, matching the AEON App wallet statement |
| cancelledAt | string | Booking cancellation time (TRAVALA only; appears after the user cancels via "Cancel Booking"; absent if not cancelled) |
| refundNote | string | Fixed statement of where the refund goes (TRAVALA only, appears after cancellation): Travel Credits go to the user's Travala account, not refunded to the platform wallet |

`receipt.shipTo` fields (SHOPIFY/AMAZON = shipping address snapshot; TRAVALA = guest, address-type fields are null):

| Parameter | SHOPIFY / AMAZON | TRAVALA |
|---|---|---|
| firstName | Recipient first name | Guest first name |
| lastName | Recipient last name | Guest last name |
| fullPhone | Recipient phone (with country code) | Guest phone |
| countryCode | Country ISO-2 code | null |
| zoneCode | State/province code | null |
| zoneName | State/province name | null |
| city | City | null |
| zip | Postal code | null |
| address1 | Street address | null |
| address2 | Additional address info | null |
| addressLine | Full concatenated display address | Hotel address |

**Response Examples**

All three channels return the **same structure** (no channel-specific fields); the difference is only in field values: see the "Channel pricing matrix" above for which line items are null, and the shipTo comparison table above for `shipTo` value differences. One example per channel follows.

SHOPIFY (COMPLETED):

```json
{
  "orderNo": "AIS178954386929718547282f",
  "outTradeNo": "MTEST1789543867",
  "channel": "SHOPIFY",
  "channelOrderId": "hWNGtEuHNQ88qSJgz5b0jDs4",
  "merchantDomain": "www.evelynlatrice.com",
  "title": "Woke Up On Mars Ebook",
  "status": "COMPLETED",
  "failCode": null,
  "reason": null,
  "addressIssue": null,
  "payNo": "AIP178954389107474263fd60",
  "pricing": {
    "subtotal": "2.00",
    "shipping": null,
    "tax": null,
    "fee": null,
    "total": "2.00",
    "currency": "USD",
    "fxRate": "1",
    "paidUsd": "2.00",
    "payMethod": "CARD",
    "payToken": null
  },
  "pendingAction": null,
  "receipt": {
    "merchantOrderNumber": "T00QW5HNJ",
    "statusUrl": null,
    "items": [
      "Product imageDescriptionQuantityPrice",
      "Quantity1Woke Up On Mars Ebook1$2.00",
      "ItemValue",
      "TotalUSD$2.00$2.00"
    ],
    "shipTo": {
      "firstName": "Waters",
      "lastName": "Alexander",
      "fullPhone": "+1 9319534025",
      "countryCode": "US",
      "zoneCode": "TN",
      "zoneName": "Tennessee",
      "city": "Knoxville",
      "zip": "37921",
      "address1": "772 Blackstock Avenue Southwest 1",
      "address2": null,
      "addressLine": "772 Blackstock Avenue Southwest 1, Knoxville, TN, 37921, US"
    },
    "checkIn": null,
    "checkOut": null,
    "txHash": null,
    "paidAt": "2026-09-16T07:31:45Z",
    "paymentRecordNo": null
  },
  "createdAt": "2026-09-16T07:31:09Z",
  "updatedAt": "2026-09-16T07:31:45Z"
}
```

AMAZON (COMPLETED) — `pricing.total`/`paidUsd` are the actually charged amount (≤ the authorization ceiling of the Create Order quote); `payMethod=CARD` and `payToken` is always null; `receipt.merchantOrderNumber` is the channel checkout order number; `shipTo` is the shipping address snapshot:

```json
{
  "orderNo": "AIS17896...",
  "outTradeNo": "MTEST1789641463",
  "channel": "AMAZON",
  "channelOrderId": "amz_22cda1e39bd74819aa692149ffbce911",
  "merchantDomain": "www.amazon.com",
  "title": "Anker USB C Hub, 7in1 Multi-Port USB Adapter, 4K@60Hz USBC to HDMI Splitter",
  "status": "COMPLETED",
  "failCode": null,
  "reason": null,
  "addressIssue": null,
  "payNo": "AIP17896...",
  "pricing": {
    "subtotal": "22.53",
    "shipping": null,
    "tax": "0.00",
    "fee": null,
    "total": "22.53",
    "currency": "USD",
    "fxRate": "1",
    "paidUsd": "22.53",
    "payMethod": "CARD",
    "payToken": null
  },
  "pendingAction": null,
  "receipt": {
    "merchantOrderNumber": "c69d7f5d-c71e-4e17-82d2-1e73a4d78645",
    "statusUrl": null,
    "items": [
      "Anker USB C Hub, 7in1 Multi-Port USB Adapter, 4K@60Hz USBC to HDMI Splitter"
    ],
    "shipTo": {
      "firstName": "Waters",
      "lastName": "Alexander",
      "fullPhone": "+1 9319534025",
      "countryCode": "US",
      "zoneCode": "TN",
      "zoneName": "Tennessee",
      "city": "Knoxville",
      "zip": "37921",
      "address1": "772 Blackstock Avenue Southwest 1",
      "address2": null,
      "addressLine": "772 Blackstock Avenue Southwest 1, Knoxville, TN, 37921, US"
    },
    "checkIn": null,
    "checkOut": null,
    "txHash": null,
    "paidAt": "2026-09-17T10:38:52Z",
    "paymentRecordNo": "WR1789641472..."
  },
  "createdAt": "2026-09-17T10:37:50Z",
  "updatedAt": "2026-09-17T10:38:52Z"
}
```

TRAVALA (COMPLETED) — `merchantDomain` is null; `receipt.merchantOrderNumber` = Travala booking number (used together with the guest's `lastName` to cancel the booking); `shipTo` is the guest snapshot (address-type fields are null, `addressLine` = hotel address); `checkIn`/`checkOut` are the stay dates:

```json
{
  "orderNo": "AIS17895...",
  "outTradeNo": "MTEST1789543100",
  "channel": "TRAVALA",
  "channelOrderId": "c58f2db4a7e94b0f9d3126aa84c1c77e",
  "merchantDomain": null,
  "title": "Ramada by Wyndham Manila Central — Classic Room, 2 Twin Beds, City View",
  "status": "COMPLETED",
  "failCode": null,
  "reason": null,
  "addressIssue": null,
  "payNo": "AIP17895...",
  "pricing": {
    "subtotal": "83.88",
    "shipping": null,
    "tax": null,
    "fee": "0.00",
    "total": "83.88",
    "currency": "USD",
    "fxRate": "1",
    "paidUsd": "83.88",
    "payMethod": "WALLET",
    "payToken": "USDT"
  },
  "pendingAction": null,
  "receipt": {
    "merchantOrderNumber": "MN5V9DWQ",
    "statusUrl": null,
    "items": [
      "Ramada by Wyndham Manila Central — Classic Room, 2 Twin Beds, City View"
    ],
    "shipTo": {
      "firstName": "Waters",
      "lastName": "Alexander",
      "fullPhone": "+8529319534025",
      "countryCode": null,
      "zoneCode": null,
      "zoneName": null,
      "city": null,
      "zip": null,
      "address1": null,
      "address2": null,
      "addressLine": "1073-1075 M.F. Jhocson Street, Sampaloc, Manila, Philippines"
    },
    "checkIn": "2026-10-02",
    "checkOut": "2026-10-04",
    "txHash": null,
    "paidAt": "2026-09-16T09:12:33Z",
    "paymentRecordNo": "WR1789540353..."
  },
  "createdAt": "2026-09-16T09:11:40Z",
  "updatedAt": "2026-09-16T09:12:33Z"
}
```

After a TRAVALA booking is cancelled via "Cancel Booking", `receipt` gains two extra fields (the order `status` remains `COMPLETED`, and no funds move on the platform side):

```json
{
  "cancelledAt": "2026-09-17T08:30:00Z",
  "refundNote": "Any refund is issued as Travala Travel Credits to the traveler's Travala account (the booking email) on travala.com, not to the platform wallet. Refund amount and timing are determined by Travala per the booking's cancellation policy."
}
```

##### Order Status `status`

| status | Meaning | Client action |
|---|---|---|
| `INIT` | Awaiting payment | Call pay to initiate payment |
| `PROCESSING` | Processing | Keep polling; if `pendingAction` is non-null, handle per "Manual Verification"; a non-null `reason` means the result is being verified — relay it truthfully; if it exceeds about 3 minutes, tell the user the merchant is slow to process |
| `COMPLETED` | Payment succeeded, item purchased | Report `receipt.merchantOrderNumber` |
| `FAIL` | Failed and closed, not purchased (funds settled or never moved) | Handle per `failCode`; display `reason` as-is |
| `TIMEOUT` | Payment authorization timed out and closed, not charged | Create a new order |

Display `reason` to the user as-is. If the reason is a merchant risk-control block, retrying at the same store will still fail; suggest switching to another store.
