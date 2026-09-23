# Card Status

**Description**

- Query the current status of the user's AI Card
- After "Initialize AI Card" is accepted, use this endpoint to poll until `ACTIVE` before performing any card fund operations (activation takes about 1–2 minutes; polling every 5–10 seconds is recommended)
- Query-only endpoint, not rate-limited

**Request**

- POST `/wallet/card/status`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for the generation rules |

**Request Example**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>" }
```

**Response Parameters**

| Parameter | Type | Description |
|---|---|---|
| cardStatus | string | AI Card status: `NOT_CREATED` not initialized (call "Initialize AI Card"); `PENDING` card creation in progress; `ACTIVE` ready to use; `UNAVAILABLE` frozen or card creation failed (contact operations) |
| failReason | string | Failure reason (returned only when `UNAVAILABLE` with a definite reason; may be shown to the user) |

**Response Example**

```json
{ "cardStatus": "ACTIVE", "failReason": null }
```
