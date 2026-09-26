# Redeem Quote

**Description**

- Query the per-transaction redeem amount range and the current network fee for the specified token + network; call this before initiating "Token Redeem",
  then validate the amount against it and display the fee to the user
- The fee fluctuates in real time with on-chain congestion; the returned value is a quote at the current moment — refresh it before redeeming and do not cache it for long

**Request**

- POST `/wallet/token/redeemQuote`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for the generation rules |
| token | Yes | string | Asset symbol |
| address | Yes | string | Receiving address (user's external wallet address) |
| network | Yes | string | Network code, taken from "Recharge & Redeem Network List" (`direction=REDEEM`) |
| amount | No | string | Estimated redeem amount (in token); when provided, the fee is calculated based on this amount |

**Request Example**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "token": "USDT","address": "TYx3p8...", "network": "TRC20", "amount": "25.5" }
```

**Response Parameters**

| Parameter | Type | Description |
|---|---|---|
| minAmount | string | Minimum redeem amount per transaction (in token) |
| maxAmount | string | Maximum redeem amount per transaction (in token) |
| networkFee | string | Current network fee (in token) |
| addressRegex | string | Address format regex (same as in "Recharge & Redeem Network List", provided here for convenience) |

**Response Example**

```json
{ "minAmount": "10", "maxAmount": "50000",
  "networkFee": "1.000000",
  "addressRegex": "^T[1-9A-HJ-NP-Za-km-z]{33}$" }
```
