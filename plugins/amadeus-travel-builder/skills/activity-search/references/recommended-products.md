# Activity search - recommended products

`POST https://api.amadeus.com/recommendation/v1/products` with `Content-Type: application/json` returns smart recommendations filtered by traveler type, interests, price, location and a similar-products seed, as a JSON array of products (schema in `product-schema.md`).

## Request body

| Field | Type | Notes |
| --- | --- | --- |
| `maxRecommendations` | int32 | Default `10`, maximum `1000` - higher values are capped. Fewer products than requested may be returned. |
| `recommendationFilters` | object | Required. Follows [recommendationFilters fields](#recommendationfilters-fields). |

## recommendationFilters fields

At least one locator is required: `iataCode`, `city`, `countryCodes`, `continentCodes` or `geoLocation`.

| Field | Type | Notes |
| --- | --- | --- |
| `iataCode` | string | Destination by IATA code. |
| `city` | string | Destination by city name. |
| `geoLocation` | `{ latitude, longitude, geoHash? }` | Destination by coordinates. Send both `latitude` and `longitude` - with one missing, the location is ignored and results come from anywhere. |
| `radius` | double | Kilometres around `geoLocation`, or around the city centre for `city`/`iataCode`. Greater than `0`. |
| `countryCodes` | `{ include: string[], exclude: string[] }` | Country codes to include or exclude, e.g. `{ "include": ["PT"] }`. Ignored next to `iataCode`, `city` or `geoLocation`; on its own, an exclude-only list searches every other country. |
| `continentCodes` | `{ include: string[], exclude: string[] }` | `EU`, `AS`, `NA`, `SA`, `AF`, `OC`. Ignored next to `iataCode`, `city`, `geoLocation` or `countryCodes.include`. |
| `interests` | string[] | Free-form phrases ("I like tapas", "art galleries", "kid-friendly"); drive semantic matching. |
| `categories` | string[] | `activities-*` taxonomy slugs, listed in `categories.md`. |
| `title` | string | Seed title for similar-products mode. |
| `minPrice` | float | Lower price bound. |
| `maxPrice` | float | Upper price bound. |
| `currency` | string | ISO 4217, upper-case, e.g. `EUR` - echoed in `priceCurrency` as sent. Without it, the price currency varies by destination. |
| `travelerType` | string | `solo` \| `family` \| `couple` \| `group`. |
| `tripContext` | string | `leisure` \| `business` \| `staycation`. |
| `travelDate` | string | ISO date (`YYYY-MM-DD`); biases results for seasonality. |
| `profile` | object | Traveler flavour flags. Follows [profile fields](#profile-fields). Doesn't change the results. |
| `risk` | string | `Safe` \| `Moderate` \| `Dangerous`. |
| `bookingEngines` | `{ include: string[], exclude: string[] }` | Booking engines to include or exclude. |
| `computationWeight` | object | Scoring weights. Fields: `title`, `price`, `geolocation`, `tripContext`, `travelerType`, `interest`, `rating`, `season`, `engagement` - all optional doubles, higher means more weight. |
| `minContentScore` | float | `0`-`1`. Doesn't change the results. |
| `varietyLevel` | string | `Low` \| `Medium` \| `High`. |

## profile fields

| Field | Type |
| --- | --- |
| `localTraveler` | boolean |
| `tourist` | boolean |
| `businessTraveler` | boolean |
| `influencer` | boolean |
| `influencerName` | string |

## Response

`200 OK` with a JSON array of products.

No pagination: `maxRecommendations` caps the result count.

## Errors

| HTTP | Code | Meaning |
| --- | --- | --- |
| 404 | 6025 | Unknown destination. |
| 404 | 6087 | Unknown `continentCodes` value. |
| 400 | 6043 | Invalid parameter value, e.g. a `categories` slug not in `categories.md`, `radius: 0` or an unsupported `currency`. |
| 500 | 6503 | Unexpected error, e.g. a negative `maxRecommendations`, a latitude outside ±90 or the deprecated `price`. |

## Old patterns

<details>
<summary>Deprecated and alternate fields</summary>

- `price` (int32) returns 500 and `priceRange` (int32) reorders results - use `minPrice` / `maxPrice`.
- `countryCode` (string) is a single-country include - use `countryCodes`.
- `currencyCode` (string) is ignored - use `currency`.

</details>
