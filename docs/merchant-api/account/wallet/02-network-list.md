# Recharge & Redeem Network List

**Description**

- Query the on-chain networks supported for a given token
- This endpoint must be called before "Get Token Recharge Address" and "Token Redeem"; `network` must be taken from the response — never spell it out yourself

**Request**

- POST `/wallet/token/networks`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for the generation rules |
| token | Yes | string | Asset symbol (e.g. `USDT` / `USDC`), taken from "Asset Balance" |
| direction | Yes | string | `RECHARGE` for recharge / `REDEEM` for redeem |

**Request Example**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "token": "USDT", "direction": "REDEEM" }
```

**Response Parameters** (per item in `networks`)

| Parameter | Type | Description |
|---|---|---|
| network | string | Network code (e.g. `TRC20` / `ERC20`); pass it to subsequent endpoints |
| networkFullName | string | Full network name (for display) |
| networkSortName | string | Short network name (for display) |
| networkLogoUrl | string | Network logo |
| memoRequired | boolean | true = transfers on this network must include a memo/tag |
| addressRegex | string | Address format regex, used to validate the receiving address before redeeming |
| networkFee | string | Network fee (in token). Returned only for `REDEEM`; null for `RECHARGE` |
| minAmount | string | Minimum redeem amount per transaction (in token). Returned only for `REDEEM` |
| maxAmount | string | Maximum redeem amount per transaction (in token). Returned only for `REDEEM` |

**Response Example**

```json
{
  "networks": [
    { "network": "TRC20",
      "networkFullName": "Tron (TRC20)", "networkSortName": "TRC20",
      "networkLogoUrl": "https://static.aeon.xyz/network/trc20.png",
      "memoRequired": false,
      "addressRegex": "^T[1-9A-HJ-NP-Za-km-z]{33}$",
      "networkFee": "1.000000", "minAmount": "10", "maxAmount": "50000" }
  ]
}
```
