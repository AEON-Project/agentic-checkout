# Product Search

**Brief Description**

- Search products / hotels by channel; SHOPIFY supports cursor-based pagination, AMAZON / TRAVALA have no pagination
- The response structure differs by channel; see the per-channel responses below

**Request Method**

- POST `/search`

**Parameters**

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier |
| userId | Yes | string | User identifier (email) |
| sign | Yes | string | Signature; see "Signature" for the generation rules |
| channel | Yes | string | `SHOPIFY` / `AMAZON` / `TRAVALA` |
| query | Yes | string | SHOPIFY/AMAZON: product keywords (English works best); TRAVALA: destination city or hotel name |
| country | SHOPIFY: Yes | string | Shipping country ISO-2 code |
| maxPrice | No | number | USD price upper limit (SHOPIFY, AMAZON). For SHOPIFY, if omitted the server-side default upper limit applies (the response carries `appliedDefaultMaxPriceUsd`) |
| minPrice | No | number | USD price lower limit (SHOPIFY only) |
| condition | No | string | `new` / `secondhand`; if omitted, both are returned (SHOPIFY only) |
| shopDomain | No | string | Restrict to a single merchant; omit for network-wide search (SHOPIFY only) |
| digitalGoods | No | boolean | Pass `true` for digital goods (e-books/software/gift cards/courses) (SHOPIFY only) |
| limit | No | number | Items per page, default 8, max 20 (SHOPIFY only) |
| cursor | No | string | Pagination cursor; omit for the first page (SHOPIFY only) |
| checkIn | TRAVALA: Yes | string | Check-in date `YYYY-MM-DD` |
| checkOut | TRAVALA: Yes | string | Check-out date `YYYY-MM-DD` |
| adults | TRAVALA: Yes | number | Number of adults |
| childrenAges | No | string | Ages of accompanying children, comma-separated, e.g. `5,7` (TRAVALA only) |
| lat | No | number | Destination latitude, improves location accuracy (TRAVALA only) |
| lng | No | number | Destination longitude, improves location accuracy (TRAVALA only) |

**Parameter Examples**

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "channel": "SHOPIFY", "query": "usb c hub", "country": "US", "maxPrice": 30 }
```

```json
{ "appId": "TEST000001", "userId": "user@example.com", "sign": "<sign>",
  "channel": "TRAVALA", "query": "Tokyo",
  "checkIn": "2026-10-12", "checkOut": "2026-10-14", "adults": 2 }
```

**Response Parameters**

##### SHOPIFY (single entry in items)

| Parameter | Type | Description |
|---|---|---|
| productId | string | Product ID; pass it to the Product Detail endpoint |
| title | string | Title (may contain `<b>` keyword highlight markers; process as needed before display) |
| description | string | Description (may be null in search results) |
| currency | string | Merchant pricing currency (ISO 4217) |
| priceMin | number | Lower bound of the variant price range, in `currency` |
| priceMax | number | Upper bound of the variant price range, in `currency` |
| merchantName | string | Merchant name (may be null) |
| merchantDomain | string | Merchant domain; must be passed to the Product Detail endpoint for single-store products |
| url | string | Product page URL |
| imageUrl | string | Main image |
| options | array | Option tree: `{name, values:[{label, available, exists}]}`; at the search stage available/exists may be null — the detail response is authoritative |
| approxCurrency | string | Currency of the approximate converted price; returned when the merchant currency is not USD, otherwise null |
| ratingScore | number | Rating (out of 5), null if unavailable |
| reviewCount | number | Number of reviews, null if unavailable |
| approxPriceMin | number | Lower bound of the approximate converted price; returned when the merchant currency is not USD, otherwise null |
| approxPriceMax | number | Upper bound of the approximate converted price; returned when the merchant currency is not USD, otherwise null |

List-level fields: `totalCount` (null when the channel cannot count), `hasMore` + `nextCursor` (cursor-based pagination),
`appliedDefaultMaxPriceUsd` (present only when the default price upper limit applies), `filteredOut` (statistics of unpurchasable products that were filtered out).

##### AMAZON (single entry in items; results without a price / temporarily unavailable have been filtered out; no pagination)

| Parameter | Type | Description |
|---|---|---|
| asin | string | Product identifier; pass it to Product Detail and Create Order |
| title | string | Title |
| price | number | Current price (USD) |
| priceStrikethrough | number | Strikethrough original price (null if there is no promotion) |
| currency | string | Currency (ISO 4217) |
| rating | number | Rating (out of 5), null if unavailable |
| reviewsCount | number | Number of reviews, null if unavailable |
| url | string | Product page URL (normalized to `https://www.amazon.com/dp/<asin>`) |
| imageUrl | string | Main image |
| isPrime | boolean | Prime badge |
| isAmazonsChoice | boolean | Amazon's Choice badge |
| bestSeller | boolean | Best Seller badge |
| pos | number | Position in search results |
| manufacturer | string | Manufacturer (may be null) |

##### TRAVALA (single entry in items; at the list level: `sessionId` — the session identifier of this search, which must be carried in both the room-package query and Create Order, and `totalFound` — total number found)

| Parameter | Type | Description |
|---|---|---|
| hotelId | string | Hotel identifier; pass it to the room-package endpoint |
| title | string | Hotel name |
| merchantName | string | Location summary (e.g. `In Tokyo (Shinjuku)`) |
| imageUrl | string | Main image |
| priceMin | number | Per-night price (USD) |
| stayTotal | number | Total price for the stay (USD) |
| currency | string | Currency, always `USD` |
| rating | number | Rating (out of 10), null if unavailable |
| stars | number | Star rating |
| reviewCount | number | Number of reviews |
| refundable | boolean | Refundable/changeable |
| freeCancelUntil | string | Free cancellation deadline (ISO-8601 UTC, may be null) |
| address | string | Hotel address |
| packageId | string | Default room package; can be used to create an order directly or switched via the room-package endpoint |

> Hotels recently observed to have no availability or an unavailable upstream supplier (the
> room-package endpoint returns an empty array, or order creation fails with `91021`) are
> automatically excluded from search results for about 30 minutes, so you never receive a
> `packageId` that is guaranteed to fail.

**Response Examples**

SHOPIFY:

```json
{
  "items": [
    {
      "productId": "gid://shopify/p/4JMypcal8df1YIQaTDQftl",
      "title": "Anker <b>332</b> USB-C Hub (5-in-1)",
      "description": "A versatile 5-in-1 hub combining USB-C, HDMI, and power delivery for fast charging and high-speed data transfer.",
      "currency": "USD",
      "priceMin": 24.99,
      "priceMax": 24.99,
      "merchantName": null,
      "merchantDomain": "us.anker.com",
      "url": "https://us.anker.com/products/a8355?variant=43216737206422",
      "imageUrl": "https://cdn.shopify.com/s/files/1/0493/9834/9974/files/A8355011_20241009_TD_926935c1-36c3-4f40-9793-09db763f1c1f.png",
      "options": [
        {
          "name": "Color",
          "values": [
            {
              "label": "Black",
              "available": null,
              "exists": null
            }
          ]
        }
      ],
      "approxCurrency": null,
      "ratingScore": null,
      "reviewCount": null,
      "approxPriceMin": null,
      "approxPriceMax": null
    }
}
```

Note: in search list items, `available`/`exists` inside `options.values` may be null (not returned by the catalog); the Product Detail endpoint gives the authoritative purchasability status. `ratingScore`/`reviewCount` are missing (null) for most products.

AMAZON:

```json
{
  "items": [
    {
      "asin": "B0CH9KN8TB",
      "title": "USB C Hub, 5-in-1 USBC to HDMI Splitter with 4K Display",
      "price": 20.99,
      "priceStrikethrough": 25.99,
      "currency": "USD",
      "rating": 4.4,
      "reviewsCount": 18800,
      "url": "https://www.amazon.com/dp/B0CH9KN8TB",
      "imageUrl": "https://m.media-amazon.com/images/I/61eSTIUmg1L._AC_UY218_.jpg",
      "isPrime": false,
      "isAmazonsChoice": false,
      "bestSeller": true,
      "pos": 14,
      "manufacturer": null
    },
    {
      "asin": "B0GWH4ZZ7T",
      "title": "Nano Laptop Docking Station Dual Monitor,8-in-1 USB C Hub for Windows",
      "price": 29.98,
      "priceStrikethrough": 35.99,
      "currency": "USD",
      "rating": 4.2,
      "reviewsCount": 301,
      "url": "https://www.amazon.com/dp/B0GWH4ZZ7T",
      "imageUrl": "https://m.media-amazon.com/images/I/71tEVIoa2UL._AC_UY218_.jpg",
      "isPrime": false,
      "isAmazonsChoice": false,
      "bestSeller": false,
      "pos": 15,
      "manufacturer": null
    }
  ]
}
```

TRAVALA:

```json
{
  "sessionId": "g3AdKu3dOrexGn6Y",
  "items": [
    {
      "hotelId": "181856",
      "title": "Hilton Tokyo Bay",
      "merchantName": "Near Tokyo DisneySea®",
      "imageUrl": "https://i.travelapi.com/lodging/1000000/190000/181900/181856/a8a67b59_b.jpg",
      "priceMin": 246.78,
      "stayTotal": 493.56,
      "currency": "USD",
      "rating": 9.0,
      "stars": 4,
      "reviewCount": 2381,
      "refundable": true,
      "freeCancelUntil": "2026-10-11T14:59:00Z",
      "address": "1-8 Maihama",
      "packageId": "A9nzIzUw2oArzV43"
    },
    {
      "hotelId": "102100364",
      "title": "Mitsui Garden Hotel Ginza Tsukiji",
      "merchantName": "Near Tsukiji Outer Market",
      "imageUrl": "https://i.travelapi.com/lodging/103000000/102110000/102100400/102100364/14b7a1bb_b.jpg",
      "priceMin": 439.93,
      "stayTotal": 879.85,
      "currency": "USD",
      "rating": 9.6,
      "stars": 4,
      "reviewCount": 702,
      "refundable": true,
      "freeCancelUntil": "2026-10-08T14:59:00Z",
      "address": "4-chome 7-1 Tsukiji",
      "packageId": "2uQ1o9LlPy7QE4Wc"
    }
  ],
  "totalFound": 8
}
```
