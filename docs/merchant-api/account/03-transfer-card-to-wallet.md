# Transfer AI Card to Wallet

**Description**

- Redeem from the AI Card to the wallet
- A network timeout does not mean failure; the request may have already been executed. Call "Asset Balance" first to verify balances before deciding whether to retry

**Request**

- POST `/wallet/card/redeem`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature, see "Signature" for generation rules |
| amountUsd | Yes | string | Redeem amount (USD), must not exceed `cardBalanceUsd` |
| token | No | string | Credited token, defaults to `USDT`. When passed explicitly, the server validates whether it is supported (rejected if unsupported, funds untouched) |
| network | No | string | Crediting network, defaults to the token's primary network. When passed explicitly, it must belong to the token's supported networks (see "Recharge & Redeem Network List", direction=RECHARGE) |

**Request Example**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "amountUsd": "5.00", "token": "USDT", "network": "BSC" }
```

**Response Parameters**

| Parameter | Type | Description |
|---|---|---|
| amountUsd | string | Echo of the redeem amount |
| token | string | Credited token |
| tokenAmount | string | Credited asset amount |
| cardBalanceUsd | string | Card balance after the redeem |

**Response Example**

```json
{ "amountUsd": "5.00", "token": "USDT",
  "tokenAmount": "5.000000", "cardBalanceUsd": "0.11" }
```
