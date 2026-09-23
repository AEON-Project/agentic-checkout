# Product Detail

**Brief Description**

- Query product / hotel details

**Request Method**

- POST `/items/detail`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for the generation rules |
| channel | Yes | string | `SHOPIFY` / `AMAZON` / `TRAVALA` |
| productId | SHOPIFY: Yes | string | `productId` from the search response |
| shopDomain | SHOPIFY: Conditional | string | Required for single-store search results; omit for network-wide search |
| country | SHOPIFY: Yes | string | Same as used in the search |
| asin | AMAZON: Yes | string | `asin` from the search response |
| hotelId | TRAVALA: Yes | string | `hotelId` from the search response |
| sessionId | TRAVALA: Yes | string | Session identifier returned by the search response |
| checkIn | TRAVALA: Yes | string | Check-in date, same as the search |
| checkOut | TRAVALA: Yes | string | Check-out date, same as the search |
| adults | TRAVALA: Yes | number | Same as the search |
| childrenAges | No | string | Same as the search (TRAVALA only) |

**Parameter Examples**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "channel": "SHOPIFY", "productId": "gid://shopify/p/4JMypcal8df1YIQaTDQftl",
  "country": "US" }
```

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "channel": "AMAZON", "asin": "B0DXJQT19B" }
```

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "channel": "TRAVALA", "hotelId": "18119", "sessionId": "g3AdKu3dOrexGn6Y",
  "checkIn": "2026-10-12", "checkOut": "2026-10-14", "adults": 2 }
```

**Response Parameters**

##### SHOPIFY

| Parameter | Type | Description |
|---|---|---|
| productId | string | Product ID |
| title | string | Title (may contain `<b>` keyword highlight markers; process as needed before display) |
| description | string | Description |
| currency | string | Merchant pricing currency (ISO 4217) |
| priceMin | number | Lower bound of the variant price range, in `currency` |
| priceMax | number | Upper bound of the variant price range, in `currency` |
| merchantName | string | Merchant name (may be null) |
| merchantDomain | string | Merchant domain |
| url | string | Product page URL |
| imageUrl | string | Main image |
| options | array | Option tree: `{name, values:[{label, available, exists}]}` |
| approxCurrency | string | Currency of the approximate converted price; returned when the merchant currency is not USD, otherwise null |
| ratingScore | number | Rating (out of 5), null if unavailable |
| reviewCount | number | Number of reviews, null if unavailable |
| approxPriceMin | number | Lower bound of the approximate converted price; returned when the merchant currency is not USD, otherwise null |
| approxPriceMax | number | Upper bound of the approximate converted price; returned when the merchant currency is not USD, otherwise null |
| images | array | Product image gallery (all image URLs); the first one is the same as `imageUrl` |
| variants | array | All purchasable variants; fields in the table below |

Single entry in `variants[]`:

| Parameter | Type | Description |
|---|---|---|
| variantId | string | Must be passed when creating an order |
| title | string | Variant name |
| sku | string | SKU (may be null) |
| url | string | Variant URL |
| merchantDomain | string | The merchant this variant belongs to; must be passed when creating an order |
| imageUrl | string | Variant image |
| price | number | Price |
| currency | string | Currency |
| available | boolean | false means not purchasable |
| options | array | `{name, label}` combination that uniquely identifies the variant |

Notes (network-wide catalog products):
- The same product may aggregate variants from **multiple merchants** (in the example below, the two variants belong to different merchants); when creating an order, always use the `variantId` + `merchantDomain` of the **selected variant**;
- The top-level `merchantDomain`/`url` in the detail response is the merchant anchored by the catalog for this request and may differ from the one in the search results — this is normal and does not affect order creation.

##### AMAZON

| Parameter | Type | Description |
|---|---|---|
| asin | string | Product identifier; must be passed when creating an order |
| parentAsin | string | Parent product identifier (present when variants exist; variants share the same parent; null otherwise) |
| title | string | Title |
| brand | string | Brand (may be null) |
| manufacturer | string | Manufacturer (may be null) |
| price | number | Current price (USD); null when the page has no price (unavailable / variant selection required) |
| priceBuybox | number | Actual buy-box price; **use this when creating an order**; null when there is no buy box |
| priceInitial | number | Strikethrough original price (present only when discounted, otherwise null) |
| currency | string | Currency (ISO 4217) |
| rating | number | Rating (out of 5), null if unavailable |
| reviewsCount | number | Number of reviews, null if unavailable |
| images | array | Image gallery |
| bulletPoints | array | List of selling points (may be null) |
| description | string | Product description (may be null) |
| availabilityText | string | Raw availability text (e.g. `In Stock` / `Currently unavailable`, may be null) |
| available | boolean | Whether an order can be created (determined by a valid price; false means an order cannot be created) |
| primeEligible | boolean | Prime shipping eligibility |
| url | string | Product page URL (normalized to `https://www.amazon.com/dp/<asin>`) |

##### TRAVALA

The response is `{ "packages": [...] }`; the table below describes a single entry in `packages[]`:

> An **empty** `packages` array means the hotel has no bookable rooms for the selected stay
> (sold out, or its upstream supplier is temporarily unavailable). This is not an error:
> choose another hotel or change the dates. The hotel will also be excluded from search
> results for about 30 minutes.

| Parameter | Type | Description |
|---|---|---|
| packageId | string | Room package to pass when creating an order |
| roomName | string | Room type name |
| mealType | string | Meals (e.g. `Room Only` / `Breakfast Included`) |
| price | number | Total price for the stay (USD), i.e. the quote used when creating an order |
| perNightUsd | number | Per-night price (USD) |
| currency | string | Currency, always `USD` |
| imageUrl | string | Room image (may be null) |
| available | boolean | Bookable (false means an order cannot be created) |
| availableRooms | number | Number of remaining rooms (may be null) |
| refundable | boolean | Refundable/changeable — cancellation policy; must be shown to the user before creating an order |
| freeCancelUntil | string | Free cancellation deadline (ISO-8601 UTC); returned only when `refundable=true`, otherwise null — cancellation policy; must be shown to the user before creating an order |
| amenities | array | Amenity/area tags (up to 12 items, may be null) |
| cancelPolicy | string | Full cancellation policy text (may span multiple lines) — cancellation policy; must be shown to the user before creating an order |

**Response Examples**

SHOPIFY:

```json
{
  "productId": "gid://shopify/p/4JMypcal8df1YIQaTDQftl",
  "title": "Anker <b>332</b> USB-C Hub (5-in-1)",
  "description": "A versatile 5-in-1 hub combining USB-C, HDMI, and power delivery for fast charging and high-speed data transfer.",
  "currency": "USD",
  "priceMin": 24.99,
  "priceMax": 24.99,
  "merchantName": null,
  "merchantDomain": "tavaone.com",
  "url": "https://tavaone.com/products/anker-b-332-b-usb-c-hub-5-in-1?variant=41949559193703",
  "imageUrl": "https://cdn.shopify.com/s/files/1/0624/5570/9799/files/A8355011_20241009_TD_926935c1-36c3-4f40-9793-09db763f1c1f.png",
  "options": [
    {
      "name": "Color",
      "values": [
        {
          "label": "Black",
          "available": true,
          "exists": true
        }
      ]
    }
  ],
  "approxCurrency": null,
  "ratingScore": null,
  "reviewCount": null,
  "approxPriceMin": null,
  "approxPriceMax": null,
  "images": [
    "https://cdn.shopify.com/s/files/1/0624/5570/9799/files/A8355011_20241009_TD_926935c1-36c3-4f40-9793-09db763f1c1f.png",
    "https://cdn.shopify.com/s/files/1/0624/5570/9799/files/Black-05_9c427f57-9d2b-4ef9-898d-e734368f2c27.jpg"
  ],
  "variants": [
    {
      "variantId": "gid://shopify/ProductVariant/41949559193703",
      "title": "Anker <b>332</b> USB-C Hub (5-in-1)",
      "sku": null,
      "url": "https://tavaone.com/products/anker-b-332-b-usb-c-hub-5-in-1?variant=41949559193703",
      "merchantDomain": "tavaone.com",
      "imageUrl": "https://cdn.shopify.com/s/files/1/0624/5570/9799/files/A8355011_20241009_TD_926935c1-36c3-4f40-9793-09db763f1c1f.png",
      "price": 24.99,
      "currency": "USD",
      "available": true,
      "options": [
        {
          "name": "Color",
          "label": "Black"
        }
      ]
    }
  ]
}
```

AMAZON:

```json
{
  "asin": "B0DXJQT19B",
  "parentAsin": "B0DXJQT19B",
  "title": "Anker USB C Hub, 7in1 Multi-Port USB Adapter, 4K@60Hz USBC to HDMI Splitter",
  "brand": "Anker",
  "manufacturer": "Anker",
  "price": 21.99,
  "priceBuybox": 21.99,
  "priceInitial": null,
  "currency": "USD",
  "rating": 4.5,
  "reviewsCount": 5224,
  "images": [
    "https://m.media-amazon.com/images/I/71Z9T0VgGyL._AC_SL1500_.jpg",
    "https://m.media-amazon.com/images/I/61-voU0mNUL._AC_SL1500_.jpg",
    "https://m.media-amazon.com/images/I/71402KGC6lL._AC_SL1500_.jpg",
    "https://m.media-amazon.com/images/I/71nKLQjF7sL._AC_SL1500_.jpg",
    "https://m.media-amazon.com/images/I/71f5iyBJ8SL._AC_SL1500_.jpg",
    "https://m.media-amazon.com/images/I/71SYhj009IL._AC_SL1500_.jpg",
    "https://m.media-amazon.com/images/I/71boMzvgX5L._AC_SL1500_.jpg",
    "https://m.media-amazon.com/images/I/61eOqIxgT-L._AC_SL1500_.jpg"
  ],
  "bulletPoints": [
    "Sleek 7-in-1 USB-C Hub: Features an HDMI port, two USB-A 3.0 ports, and a USB-C data port, each providing 5Gbps transfer speeds. It also includes a USB-C PD input port for charging up to 100W and dual SD and TF card slots, all in a compact design.",
    "Flawless 4K@60Hz Video with HDMI: Delivers exceptional clarity and smoothness with its 4K@60Hz HDMI port, making it ideal for high-definition presentations and entertainment. (Note: Only the HDMI port supports video projection; the USB-C port is for data transfer only.)",
    "Double Up on Efficiency: The two USB-A 3.0 ports and a USB-C port support a fast 5Gbps data rate, significantly boosting your transfer speeds and improving productivity.",
    "Fast and Reliable 85W Charging: Offers high-capacity, speedy charging for laptops up to 85W, so you spend less time tethered to an outlet and more time being productive.",
    "What You Get: Anker USB-C Hub (7-in-1), welcome guide, 18-month warranty, and our friendly customer service."
  ],
  "description": "Anker 7-in-1 USB C Hub\nAnker 8-in-1 USB C Hub\nAnker 7-in-2 USB C Hub\nAnker 5-in-1 USB C Hub\nAnker 5-in-1 USB C Hub\n6-in-1 USB-C Hub\nAnker 553 Hub 8-in-1",
  "availabilityText": "In Stock",
  "available": true,
  "primeEligible": true,
  "url": "https://www.amazon.com/dp/B0DXJQT19B"
}
```

TRAVALA:

```json
{
  "packages": [
    {
      "packageId": "ydJYmUTT94xvacGW",
      "roomName": "Room, 1 King Bed (High Floor)",
      "mealType": "Room Only",
      "price": 1117.12,
      "perNightUsd": 558.56,
      "currency": "USD",
      "imageUrl": "https://i.travelapi.com/lodging/1000000/20000/18200/18119/e56fbdc6_z.jpg",
      "available": true,
      "availableRooms": 13,
      "refundable": true,
      "freeCancelUntil": "2026-10-11T14:59:00Z",
      "amenities": [
        "Room service (limited hours)",
        "Refrigerator",
        "Daily housekeeping",
        "Phone",
        "Espresso maker"
      ],
      "cancelPolicy": "Free cancellation until 11 October 2026 14:59 (GMT+00:00).\nCancellations between 11 October 2026 14:59 (GMT+00:00) and 12 October 2026 14:59 (GMT+00:00) will result in a 1 night penalty charge.\nThe end time for the cancellation window is 12 October 2026 14:59 (GMT+00:00) at which time the booking will become fully non-refundable.\nIf you fail to check-in for this reservation, or if you cancel or change this reservation after check-in, you may incur penalty charges at the discretion of the property of up to 100% of the booking value."
    }
  ]
}
```
