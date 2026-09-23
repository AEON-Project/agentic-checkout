# Token Redeem

**Description**

- Redeem wallet tokens to the user's own external address: once accepted, the payout is sent on-chain and a redeem transaction number is returned
- The fee is calculated by the server in real time per network and settled out of the redeem amount; the actual amount received is subject to the on-chain result. Before initiating,
  validate the amount via "Redeem Quote", display `networkFee` to the user, and get their confirmation
- The receiving address and memo are the merchant's responsibility to verify (use `addressRegex` for pre-validation): on-chain transfers with a wrong address or a missing memo
  cannot be reversed
- A network timeout does not mean failure: first check the balance via "Asset Balance", then decide whether to resend

**Request**

- POST `/wallet/token/redeem`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for the generation rules |
| token | Yes | string | Asset symbol |
| network | Yes | string | Network code, taken from "Recharge & Redeem Network List" (`direction=REDEEM`) |
| address | Yes | string | Receiving address (user's external wallet address) |
| memo | Conditional | string | Required for networks with `memoRequired=true` |
| amount | Yes | string | Redeem amount (in token); must be between `minAmount` and `maxAmount` from "Redeem Quote" |

**Request Example**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "token": "USDT", "network": "TRC20",
  "address": "TYx3p8...", "memo": "", "amount": "25.5" }
```

**Response Parameters**

| Parameter | Type | Description |
|---|---|---|
| redeemNo | string | Redeem transaction number (for reconciliation / customer support) |
| token | string | Token |
| network | string | Network |
| amount | string | Redeem amount echoed back |
| networkFee | string | Network fee for this transaction (in token) |
| status | string | Status after acceptance: `PENDING` — on-chain payout in progress |

**Response Example**

```json
{ "redeemNo": "W17889...", "token": "USDT", "network": "TRC20",
  "amount": "25.500000", "networkFee": "1.000000", "status": "PENDING" }
```
