# Card Transactions (Recharge / Redeem)

**Description**

- Paginated query of the AI Card's inbound (recharge) / outbound (redeem) records, in reverse chronological order

**Request**

- POST `/wallet/card/records`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for the generation rules |
| direction | Yes | string | `TOPUP` into card (recharge) / `REDEEM` out of card (redeem) |
| page | No | number | Page number, starting from 1; default 1 |
| pageSize | No | number | Items per page; default 20, max 50 |

**Request Example**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "direction": "TOPUP", "page": 1, "pageSize": 20 }
```

**Response Parameters** (per item in `list`. List level: `list` + `page` / `pageSize` / `total`)

| Parameter | Type | Description |
|---|---|---|
| recordNo | string | Transaction record number |
| token | string | Transferred asset |
| network | string | Network of the transferred asset |
| tokenAmount | string | Asset amount |
| usdAmount | string | Equivalent USD amount (card-side credit/debit amount) |
| status | string | `COMPLETED` completed / `PROCESSING` in progress / `FAILED` failed |
| createdAt | string | Creation time |

**Response Example**

```json
{
  "list": [
    { "recordNo": "AGTU17894...", "token": "USDT", "network": "BSC",
      "tokenAmount": "5.000000", "usdAmount": "5.00",
      "status": "COMPLETED", "createdAt": "2026-09-15T04:10:02Z" }
  ],
  "page": 1, "pageSize": 20, "total": 1
}
```
