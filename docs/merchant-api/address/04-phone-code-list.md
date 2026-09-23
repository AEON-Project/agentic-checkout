# Phone Code List

**Description**

- Call before saving an address; returns phone country code dropdown data

**Request**

- POST `/address/phoneCodes`

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
| code | string | Country ISO-2 code |
| nameEn | string | English name |
| nameZh | string | Chinese name |
| phoneCode | string | Phone country code (without +); pass it when saving an address |
| flag | string | Flag emoji |

**Response Example**

```json
[
  { "code": "US", "nameEn": "United States", "nameZh": "美国",
    "phoneCode": "1", "flag": "🇺🇸" }
]
```
