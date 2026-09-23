# Save Address

**Description**

- Save/overwrite the user's single shipping address
- Country and state/province must use the code values returned by the "Country List" and "State & Province List" APIs

**Request**

- POST `/address/save`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature, see "Signature" for generation rules |
| lastName | Yes | string | Recipient last name |
| firstName | Yes | string | Recipient first name |
| phoneCode | Yes | string | Phone country code (without +), taken from "Phone Code List" |
| phone | Yes | string | Phone number (without country code) |
| countryCode | Yes | string | Country ISO-2 code, taken from "Country List" |
| zoneCode | Conditional | string | State/province code, taken from "State & Province List"; may be omitted when the country's regions list is empty |
| address1 | Yes | string | Street address |
| address2 | No | string | Apartment/unit supplement |
| city | Yes | string | City |
| zip | Yes | string | ZIP/postal code |

**Request Example**

```json
{
  "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "lastName": "Alexander", "firstName": "Waters",
  "phoneCode": "1", "phone": "9319534025",
  "countryCode": "US", "zoneCode": "TN",
  "address1": "772 Blackstock Avenue Southwest", "address2": "",
  "city": "Knoxville", "zip": "37921"
}
```

**Response Parameters**

| Parameter | Type | Description |
|---|---|---|
| saved | boolean | Saved successfully |

**Response Example**

```json
{ "saved": true }
```
