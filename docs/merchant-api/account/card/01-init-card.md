# Initialize AI Card

**Description**

- Creates an AI Card for the user by deducting the specified token from the user's wallet;
  the amount is converted to USD at the real-time exchange rate as the card creation amount. The converted value must be at least **0.6 USD** — anything less is rejected outright without touching funds.
- Card creation is an asynchronous flow: this endpoint returns `PENDING` upon acceptance; use "Card Status" (`/wallet/card/status`)
  to poll for confirmation (activation takes about 1–2 minutes).
- Idempotent: if the user already has a card (including one being created or failed), the endpoint returns its current status directly and **never deducts funds again**;
  while the outcome of a previous card creation deduction is still undetermined (network timeout, etc.), the server rejects any new request outright, eliminating duplicate deductions.
- Fund protection: if the deduction or card creation definitively fails, the deducted assets are automatically refunded to the wallet; if the outcome is unknown, no final state is set —
  call "Card Status" to confirm first, then decide whether to retry. Do not blindly resubmit.
- The SHOPIFY channel pays with the card; this endpoint must be completed before any payment or transfer into the card;

**Request**

- POST `/wallet/card/init`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for the generation rules |
| token | Yes | string | Deduction token name, e.g. `USDT` |
| amount | Yes | string | Deduction token amount (converted to USD at the real-time exchange rate; the converted value must be at least 0.6 USD) |

**Request Example**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "token": "USDT", "amount": "1" }
```

**Response Parameters**

| Parameter | Type | Description |
|---|---|---|
| cardStatus | string | AI Card status: `PENDING` card creation in progress (normal acceptance result); `ACTIVE` ready to use (idempotent hit on an existing card); `UNAVAILABLE` frozen or card creation failed (contact operations) |

**Response Example**

```json
{ "cardStatus": "PENDING" }
```

**Common Failures**

| Scenario | Error code | Fund semantics |
|---|---|---|
| Converted value below 0.6 USD / invalid parameters | 91017 | Funds untouched |
| Insufficient wallet balance, deduction definitively failed | 91010 | Funds untouched |
| Card creation failed (deducted funds automatically refunded to wallet) | 91010 | Automatically refunded |
| Outcome unconfirmed (network timeout, etc.) | 1 | Funds may have been deducted; call "Card Status" to confirm first, then decide |
