# Manual Verification (3DS)

**Summary**

- Some checkouts require a one-time verification code: when the order detail stays `PROCESSING` and `pendingAction` becomes non-null, ask the user for the verification code and submit it via this endpoint
- Valid for about 3 minutes

`pendingAction` structure (from the "Payment Detail" response):

```json
"pendingAction": { "type": "3DS_OTP",
                   "hint": "Enter the verification code sent to the cardholder",
                   "expiresAt": "2026-09-09T17:15:00Z" }
```

**Request**

- POST `/orders/actions`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for generation rules |
| orderNo | Yes | string | System order number returned by the pay response |
| type | Yes | string | Action type; echo back `pendingAction.type` (e.g. `3DS_OTP`) |
| value | Yes | string | User input value (e.g. the verification code) |

**Request Example**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "orderNo": "AIS17889...", "type": "3DS_OTP", "value": "123456" }
```

**Response Parameters**

| Parameter | Type | Description |
|---|---|---|
| accepted | boolean | Accepted; continue polling |

**Response Example**

```json
{ "accepted": true }
```
