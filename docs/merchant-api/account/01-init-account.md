# Initialize Account

**Description**

- Provision an account for the user: creates the account

**Request**

- POST `/account/init`

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
| initialized | boolean | Account and wallet are ready |
| cardStatus | string | AI Card status (read-only): `NOT_CREATED` not initialized (call "Initialize AI Card"); `ACTIVE` ready to use; `PENDING` card creation in progress; `UNAVAILABLE` frozen or card creation failed (contact operations) |

**Response Example**

```json
{ "initialized": true, "cardStatus": "NOT_CREATED" }
```
