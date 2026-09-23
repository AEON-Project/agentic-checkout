# Wallet Transactions

**Description**

- Paginated query of the user's wallet transaction records, in reverse chronological order;

**Request**

- POST `/wallet/records`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for the generation rules |
| type | No | string | Record type: `DEPOSIT` on-chain recharge / `WITHDRAW` token redeem / `REFUND` refund / `AI_TRANSFER` wallet-to-card transfer / `AI_REDEEM` card-to-wallet redeem; omit for all |
| token | No | string | Filter by asset symbol (e.g. `USDT`) |
| page | No | number | Page number, starting from 1; default 1 |
| pageSize | No | number | Items per page; default 20, max 50 |

**Request Example**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "type": "WITHDRAW", "token": "USDT", "page": 1, "pageSize": 20 }
```

**Response Parameters** (per item in `list`; for records not related to the chain, network/address/txHash/memo are null.
List level: `list` + `page` / `pageSize` / `total`)

| Parameter | Type | Description |
|---|---|---|
| recordNo | string | Transaction record number (for reconciliation / customer support) |
| type | string | Record type (same enum as the request `type`) |
| token | string | Asset symbol |
| amount | string | Amount (in token) |
| network | string | On-chain network |
| address | string | On-chain address (recharge = crediting address, redeem = receiving address) |
| txHash | string | On-chain transaction hash (verifiable on a block explorer); null when not yet on-chain / in progress |
| memo | string | Transfer memo/tag |
| status | string | `PENDING` in progress / `SUCCESS` success / `FAIL` failed |
| createdAt | string | Creation time |
| updatedAt | string | Last update time |

**Response Example**

```json
{
  "list": [
    { "recordNo": "W17889...", "type": "WITHDRAW",
      "token": "USDT", "amount": "25.500000",
      "network": "TRC20", "address": "TYx3p8...",
      "txHash": "8a4f...", "memo": null,
      "status": "SUCCESS",
      "createdAt": "2026-09-15T04:20:00Z", "updatedAt": "2026-09-15T04:25:31Z" }
  ],
  "page": 1, "pageSize": 20, "total": 3
}
```
