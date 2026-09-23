# State & Province List

**Description**

- Call before saving an address; returns state/province dropdown data for the specified country; an empty list means the country does not require a state/province

**Request**

- POST `/address/regions`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature, see "Signature" for generation rules |
| country | Yes | string | Country ISO-2 code |

**Request Example**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>", "country": "US" }
```

**Response Parameters**

| Parameter | Type | Description |
|---|---|---|
| zoneCode | string | State/province code; pass it when saving an address |
| nameEn | string | English name |
| nameZh | string | Chinese name |
| provinceKey | string | Subdivision designation (`STATE` / `PROVINCE` / `REGION`…), for form labels |

**Response Example**

```json
[
  { "zoneCode": "TN", "nameEn": "Tennessee", "nameZh": "田纳西州",
    "provinceKey": "STATE" }
]
```
