# Get Token Recharge Address

**Description**

- Get the user's dedicated recharge address for the specified token + network; it is generated automatically on the first call and remains fixed afterwards
- Transfers sent to this address are credited to the wallet (crediting is subject to on-chain confirmation; once credited, the funds are visible via "Asset Balance")
- You must emphasize to the user: only transfer the selected token on the selected network — assets sent on the wrong network or with the wrong token cannot be recovered;
  for networks with `memoRequired=true`, the `memo` must be displayed together with the address — transfers missing the memo will be lost

**Request**

- POST `/wallet/token/rechargeAddress`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for the generation rules |
| token | Yes | string | Asset symbol |
| network | Yes | string | Network code, taken from "Recharge & Redeem Network List" (`direction=RECHARGE`) |

**Request Example**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "token": "USDT", "network": "TRC20" }
```

**Response Parameters**

| Parameter | Type | Description |
|---|---|---|
| address | string | Recharge address |
| memo | string | Recharge memo/tag; null when `memoRequired=false` |
| memoRequired | boolean | Whether the transfer must carry a memo |

**Response Example**

```json
{ "address": "TXk8m2...", "memo": null, "memoRequired": false }
```
