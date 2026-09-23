# Card Consumption Records

**Description**

- Paginated query of the AI Card's consumption transaction records (checkout charges), in reverse chronological order

**Request**

- POST `/wallet/card/transactions`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for the generation rules |
| page | No | number | Page number, starting from 1; default 1 |
| pageSize | No | number | Items per page; default 20, max 50 |

**Request Example**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "page": 1, "pageSize": 20 }
```

**Response Parameters** (per item in `list`. List level: `list` + `page` / `pageSize` / `total`)

| Parameter | Type | Description |
|---|---|---|
| transactionId | string | Card transaction number |
| merchantName | string | Merchant name (display name on the acquiring side) |
| amount | string | Transaction amount |
| currency | string | Transaction currency |
| status | string | `COMPLETED` completed / `REFUND` refunded / `PROCESSING` in progress / `FAILED` failed |
| transactionTime | string | Transaction time |

**Response Example**

```json
{
  "list": [
    { "transactionId": "TXN9f2c...", "merchantName": "Acme Gear",
      "amount": "26.09", "currency": "USD",
      "status": "COMPLETED", "transactionTime": "2026-09-09T17:12:30Z" },
    { "transactionId": "TXN8e1a...", "merchantName": "Downpour Books",
      "amount": "0.54", "currency": "USD",
      "status": "REFUND", "transactionTime": "2026-09-15T04:18:47Z" }
  ],
  "page": 1, "pageSize": 20, "total": 12
}
```
