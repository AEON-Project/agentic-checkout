# Country List

**Description**

- Call before saving an address; returns country dropdown data (only countries where shipping is supported)

**Request**

- POST `/address/countries`

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
| code | string | Country ISO-2 code; pass it when saving an address |
| nameEn | string | English name |
| nameZh | string | Chinese name |
| phoneCode | string | Phone country code (without +) |
| flag | string | Flag emoji |

**Response Example**

```json
[
  { "code": "US", "nameEn": "United States", "nameZh": "美国",
    "phoneCode": "1", "flag": "🇺🇸" }
]
```
