# Transfer Wallet to AI Card

**Description**

- Transfer wallet assets into the card.
- A network timeout does not mean failure; the request may have already been executed. Call "Asset Balance" first to verify balances before deciding whether to retry

**Request**

- POST `/wallet/card/topup`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature, see "Signature" for generation rules |
| amountUsd | Yes | string | Amount to load onto the card (USD) |
| token | No | string | Wallet asset to debit; if omitted, chosen by the server |

**Request Example**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "amountUsd": "20.00", "token": "USDT" }
```

**Response Parameters**

| Parameter | Type | Description |
|---|---|---|
| amountUsd | string | Echo of the loaded amount |
| token | string | Token actually debited |
| tokenAmount | string | Asset amount actually debited |
| cardBalanceUsd | string | Card balance after the transfer |

**Response Example**

```json
{ "amountUsd": "20.00", "token": "USDT",
  "tokenAmount": "20.000000", "cardBalanceUsd": "20.11" }
```
