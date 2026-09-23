# Order List

**Brief Description**

- Paginated query of the user's orders, sorted by creation time in descending order

**Request Method**

- POST `/orders/list`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for the generation rules |
| status | No | string | Filter by status: `INIT` / `PROCESSING` / `COMPLETED` / `FAIL` / `TIMEOUT` |
| from | No | string | Start of the creation date range `YYYY-MM-DD` (UTC, inclusive) |
| to | No | string | End of the creation date range `YYYY-MM-DD` (UTC, inclusive) |
| page | No | number | Page number, starting at 1, default 1 |
| pageSize | No | number | Items per page, default 20, max 50 |

**Parameter Example**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "status": "COMPLETED", "from": "2026-09-01", "to": "2026-09-09", "page": 1, "pageSize": 20 }
```

**Response Parameters**

| Parameter | Type | Description |
|---|---|---|
| list | array | Order list; see the table below for per-item fields |
| page | number | Page number |
| pageSize | number | Items per page |
| total | number | Total count |

Per-item fields of `list` (same as "Payment Detail", without `pendingAction`):

| Parameter | Type | Description |
|---|---|---|
| orderNo | string | Platform order number |
| channel | string | Channel |
| channelOrderId | string | Channel order ID |
| outTradeNo | string | Merchant external order number (the value passed at payment time is echoed back as-is; the merchant uses it to match its own order) |
| merchantDomain | string | Merchant domain (SHOPIFY) |
| title | string | Product/booking title (same as `quote.title` from Create Order) |
| status | string | Order status: `INIT` awaiting payment / `PROCESSING` processing / `COMPLETED` payment succeeded / `FAIL` failed and closed / `TIMEOUT` payment authorization timed out and closed |
| failCode | string | Failure code, non-null only when `status=FAIL`; values are the same set as "Error Codes" |
| reason | string | End-user-facing failure reason in English |
| payNo | string | Latest payment order number (reconciliation evidence); null if no payment was made |
| pricing | object | Confirmed real cost breakdown and actual charge; see the table below for fields |
| receipt | object | Success receipt, non-null only when `COMPLETED`; see the table below for fields |
| createdAt | string | Order creation time |
| updatedAt | string | Latest status change time |

`pricing` fields (all amounts are strings; line items the channel does not have are null):

| Parameter | Type | Description |
|---|---|---|
| currency | string | Pricing currency (ISO 4217) |
| subtotal | string | Product subtotal |
| shipping | string | Shipping cost |
| tax | string | Tax |
| fee | string | Service fee |
| total | string | Real payable total = sum of the non-null line items |
| fxRate | string | `currency` to USD exchange rate (`"1"` when `currency=USD`) |
| paidUsd | string | Actual charged amount (USD), = `total` × `fxRate` |
| payMethod | string | Payment method of this order: `CARD` / `WALLET` |
| payToken | string | Asset actually debited on the wallet side; null if the card balance was sufficient and the wallet was not touched |

`receipt` fields (items the channel cannot provide are null):

| Parameter | Type | Description |
|---|---|---|
| merchantOrderNumber | string | Merchant-side order credential: SHOPIFY = merchant confirmation number (e.g. `#1042`); AMAZON = checkout order number (anchor for status query/reconciliation); TRAVALA = booking number |
| statusUrl | string | Merchant order status page link (no login required; check shipping/status afterwards). SHOPIFY only |
| items | array | Purchased content lines |
| shipTo | object | Shipping/guest snapshot (the actual delivery information at payment time); see the table below for fields |
| checkIn | string | Check-in date (TRAVALA only, `YYYY-MM-DD`) |
| checkOut | string | Check-out date (TRAVALA only, `YYYY-MM-DD`) |
| txHash | string | On-chain payout transaction hash (legacy field, always null for all channels currently) |
| paidAt | string | Payment completion time |
| paymentRecordNo | string | Platform charge record number, matching the AEON App wallet statement |

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

**Response Example**

```json
{
  "list": [
    {
      "orderNo": "AIS17889...", "channel": "SHOPIFY", "channelOrderId": "hWNGc...",
      "outTradeNo": "M20260915170001",
      "merchantDomain": "acmegear.com",
      "title": "4-in-1 USB-C Hub — Space Gray",
      "status": "COMPLETED",
      "failCode": null,
      "reason": null,
      "payNo": "AIP17889...",
      "pricing": { "subtotal": "19.99", "shipping": "4.90", "tax": "1.20", "fee": null,
                   "total": "26.09", "currency": "USD", "fxRate": "1",
                   "paidUsd": "26.09", "payMethod": "CARD", "payToken": "USDT" },
      "receipt": { "merchantOrderNumber": "#1042",
                   "statusUrl": "https://acmegear.com/.../orders/...",
                   "items": ["4-in-1 USB-C Hub — Space Gray ×1"],
                   "shipTo": { "firstName": "Waters", "lastName": "Alexander",
                               "fullPhone": "+1 9319534025",
                               "countryCode": "US", "zoneCode": "TN", "zoneName": "Tennessee",
                               "city": "Knoxville", "zip": "37921",
                               "address1": "772 Blackstock Avenue Southwest", "address2": null,
                               "addressLine": "772 Blackstock Avenue Southwest, Knoxville, TN 37921, US" },
                   "checkIn": null, "checkOut": null, "txHash": null,
                   "paidAt": "2026-09-09T17:12:30Z",
                   "paymentRecordNo": "WR17889..." },
      "createdAt": "2026-09-09T17:10:02Z",
      "updatedAt": "2026-09-09T17:12:41Z"
    }
  ],
  "page": 1, "pageSize": 20, "total": 57
}
```
