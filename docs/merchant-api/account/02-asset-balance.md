# Asset Balance

**Description**

- Query the balances of each wallet asset and the AI Card balance; check balances before paying, and guide the user to recharge when insufficient (see "Get Token Recharge Address")

**Request**

- POST `/wallet/assets`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature, see "Signature" for generation rules |

**Request Example**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>" }
```

**Response Parameters**

| Parameter | Type | Description |
|---|---|---|
| assets[].token | string | Asset symbol (e.g. `USDT` / `USDC`), network-agnostic |
| assets[].balance | string | Balance (in the asset's native precision) |
| assets[].usdValue | string | Value in USD |
| cardBalanceUsd | string | AI Card balance (USD) |

**Response Example**

```json
{
  "assets": [
    { "token": "USDT", "balance": "25.310000", "usdValue": "25.31" },
    { "token": "USDC", "balance": "4.000000",  "usdValue": "4.00" }
  ],
  "cardBalanceUsd": "0.11"
}
```
