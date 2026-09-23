# Get Address

**Description**

- Query the user's saved shipping address

**Request**

- POST `/address/get`

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
| firstName | string | Recipient first name |
| lastName | string | Recipient last name |
| phoneCode | string | Phone country code (without +) |
| phone | string | Phone number |
| fullPhone | string | Concatenated display string (with +) |
| countryCode | string | Country ISO-2 code |
| zoneCode | string | State/province code; null for countries without states/provinces |
| zoneName | string | State/province name; null for countries without states/provinces |
| address1 | string | Street address |
| address2 | string | Apartment/unit supplement (nullable) |
| city | string | City |
| zip | string | ZIP/postal code |

**Response Example**

```json
{
  "firstName": "Waters", "lastName": "Alexander",
  "phoneCode": "1", "phone": "9319534025", "fullPhone": "+1 9319534025",
  "countryCode": "US", "zoneCode": "TN", "zoneName": "Tennessee",
  "address1": "772 Blackstock Avenue Southwest", "address2": null,
  "city": "Knoxville", "zip": "37921"
}
```
