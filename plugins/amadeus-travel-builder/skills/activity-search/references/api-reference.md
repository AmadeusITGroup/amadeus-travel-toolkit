# Activity search - API reference

Detailed reference for `GET /recommendation/v1/top-activities`, `POST /recommendation/v1/products` and `GET /api/consumer/product/{productId}`. The base URL and authentication model (`X-API-Key` header from `AMADEUS_API_KEY`) are documented in [`../SKILL.md`](../SKILL.md#gateway-and-authentication).

## Endpoints

```
GET  https://api.amadeus.com/recommendation/v1/top-activities
POST https://api.amadeus.com/recommendation/v1/products
GET  https://api.amadeus.com/api/consumer/product/{productId}
```

## Top activities - `GET /recommendation/v1/top-activities`

Retrieve the top-performing products at a destination.

### Query parameters

| Parameter | Required | Type | Notes |
|---|---|---|---|
| `iataCode` | see note | string | City / airport IATA. At least one location param required; see priority note below. |
| `cityName` | see note | string | City name. Lowest priority when multiple location params provided. |
| `lat` | see note | double | Latitude. Pair with `lon` for geo-based location. Second-highest priority. |
| `lon` | see note | double | Longitude. Pair with `lat` for geo-based location. |
| `radius` | optional | double | Radius in km around `lat`/`lon`. |
| `maxRecommendations` | optional | int32 | Cap on number of results. |
| `computationWeight` | optional | string | Stringified scoring config (see `ComputationWeight` schema below). |
| `minContentScore` | optional | float (0–1) | Filter out low-content products. |
| `currency` | optional | ISO 4217 | E.g. `EUR`, `USD`. |

### Response

`200 OK` returns a **JSON array** of `GetProductResponse` objects.

`404` is returned when no products match.

## Recommended products - `POST /recommendation/v1/products`

Smart recommendations filterable by traveler type, interests, price, geo, and similar-products seed.

### Request body - `FindRecommendationsConsumerRequest`

```json
{
  "maxRecommendations": 10,
  "recommendationFilters": { /* see below */ }
}
```

### `recommendationFilters` - `RecommendationConsumerFilterRequest`

| Field | Type | Notes |
|---|---|---|
| `city` | string | Destination by city name. At least one of `iataCode`, `city`, `countryCodes`, `continentCodes`, or `geoLocation` is required. |
| `iataCode` | string | Destination by IATA code. |
| `geoLocation` | `{ latitude, longitude, geoHash? }` | Destination by coordinates. |
| `radius` | double | Radius in km around `geoLocation`. |
| `countryCode` | string | Single country (deprecated in favour of `countryCodes`). |
| `countryCodes` | `IncludeExcludeField` | `{ include: string[], exclude: string[] }`. |
| `continentCodes` | `IncludeExcludeField` | Same shape - continent codes. |
| `categories` | string[] | Taxonomy codes. No lookup endpoint exists - see Behaviour notes before using this instead of `interests`. |
| `interests` | string[] | Free-form interest phrases. |
| `title` | string | Seed title for similar-products mode. |
| `minPrice` | float | Lower price bound. |
| `maxPrice` | float | Upper price bound. |
| `price` | int32 | **Deprecated** - use `minPrice`/`maxPrice`. |
| `priceRange` | int32 | **Deprecated** - use `minPrice`/`maxPrice`. |
| `currency` | string | ISO 4217. |
| `currencyCode` | string | Alternate spelling - prefer `currency`. |
| `profile` | `RecommendationProfile` | Boolean flavour flags + optional `influencerName`. |
| `risk` | string | `Safe` \| `Moderate` \| `Dangerous`. |
| `tripContext` | string | `leisure` \| `business` \| `staycation`. |
| `travelerType` | string | `solo` \| `family` \| `couple` \| `group`. |
| `travelDate` | string | ISO date - biases for seasonality. |
| `bookingEngines` | `IncludeExcludeField` | Filter on booking engine. |
| `computationWeight` | `ComputationWeight` | Tune scoring weights - see schema. |
| `minContentScore` | float (0–1) | |
| `varietyLevel` | string | `Low` \| `Medium` \| `High`. |

### `RecommendationProfile`

| Field | Type |
|---|---|
| `localTraveler` | boolean |
| `tourist` | boolean |
| `businessTraveler` | boolean |
| `influencer` | boolean |
| `influencerName` | string |

### `ComputationWeight`

All fields are doubles, all optional. Higher = more weight in the score.

| Field |
|---|
| `title` |
| `price` |
| `geolocation` |
| `tripContext` |
| `travelerType` |
| `interest` |
| `rating` |
| `season` |
| `engagement` |

### `IncludeExcludeField`

```json
{ "include": ["ES", "PT"], "exclude": [] }
```

### Response

`200 OK` returns a **JSON array** of `GetProductResponse` objects.

## Get product - `GET /api/consumer/product/{productId}`

Retrieve full detail for a single, already-known product.

Note the path: this endpoint is `/api/consumer/product/{productId}`, not under `/recommendation/v1` like the other two.

### Path parameters

| Parameter | Required | Type | Notes |
|---|---|---|---|
| `productId` | yes | int64 | The catalogue `productId` from a prior top-activities / recommended-products result (or a stored favourite / deep link). |

### Response

`200 OK` returns a **single `GetProductResponse` object** - not wrapped in an array. `id` and `productId` come back equal on this endpoint, since there's no search context to diverge them.

`404` is returned when `productId` doesn't exist or is no longer available.

## `GetProductResponse` schema

| Field | Type | Notes |
|---|---|---|
| `id` | int64 | Dynamic per call - encodes the context of the search that returned the product. Not a stable identifier, and may exceed 53-bit float precision. |
| `productId` | int64 | Static ID of the product in the catalogue. Use this for identity, caching and dedupe. |
| `externalId` | string | Booking-engine-specific external ID. |
| `title` | string | Display title. |
| `shortDescription` | string | One-line teaser. Optional - omitted on most products. |
| `description` | string | Full description - **may contain HTML**. Sanitize before rendering. |
| `thumbnailImage` | string | Image identifier or URL. |
| `galleryImages` | `Image[]` | `{ title?, url, order }`. |
| `bookingEngine` | string | E.g. `TIQETS`, `HOLIBOB_TIQETS`, `GETYOURGUIDE`. |
| `bookingEngineName` | string | Display name. |
| `bookingUrl` | string | URL to redirect the user to for booking. |
| `travelerUrl` | string | Tracking-wrapped deep link to the traveler-facing product page; distinct from `bookingUrl`. |
| `minPrice` / `maxPrice` | double | |
| `priceCurrency` | string | |
| `pricingUnit` | string | E.g. `PER_PERSON`, `NONE`. |
| `minParticipants` / `maxParticipants` | int32 | Optional - omitted on most products. |
| `address` | string | Free-text address. |
| `geoLocation` | `{ latitude, longitude, geoHash? }` | |
| `continentCode` | string | Only present on `/recommendation/v1/products` results, e.g. `"EU"`. Top-activities and get-product never return it, even for the same product. |
| `duration` | string | Free-text duration ("2 hours"). |
| `durationHours` | double | Numeric duration when known. |
| `cityName` / `countryName` / `zipCode` | string | `zipCode` is a postal code (e.g. `"78000"`), present only when the venue has one. |
| `categories` | `Category[]` | `{ level, code, name?, description? }`. |
| `taxonomies` | `Taxonomy[][]` | 2D - each inner array is one hierarchical taxonomy path. |
| `descriptors` | `{ type, reason }[]` | Marketing tags, e.g. `[{ "type": "PROMOTED", "reason": "PROMOTED" }]`. Empty on nearly all products. |
| `availableLanguages` | string[] | ISO language codes. |
| `displayedLanguage` | string | Single ISO code - the language `description`/`title` are written in. |
| `guideLanguages` | `{ live?: string[] }` | Languages a live guide is offered in, where applicable. |
| `rating` | double | 0–5. |
| `numberOfRatings` | int32 | |
| `bestseller` | boolean | |
| `instantConfirmation` | boolean | |
| `instantDelivery` | boolean | |
| `skipTheLine` | boolean | |
| `onlineBookable` | boolean | |
| `isCancellable` | boolean | |
| `cancellationCutOff` | string | Free-text cancellation policy, e.g. "full refund if more than 24h before start." Distinct from the structured fields below. |
| `cancellationCutOffAmount` / `cancellationCutOffUnit` | int32 / string | |
| `modalitiesIncluded` / `modalitiesExcluded` / `modalitiesAdditionalInfo` | string | Free-text inclusions/exclusions. |
| `experienceHighlights` | string | Free-text bullet-list-ish. |
| `itinerary` | `{ type, text }` | `type` seen as `UNSTRUCTURED`; free-text itinerary description. Whether it's present depends on the product, not the endpoint - the same product returns it (or omits it) the same way from top-activities, recommended-products and get-product alike. |
| `additionalInformation` | object | Free-form key/value map (e.g. `"Meeting Point"`) plus an `unstructured_data` string of newline-joined notices. Keys vary per product. |
| `availabilityType` | string | E.g. `OPENING_HOURS`. |
| `availabilityPeriods` | `{ weeklyTimes?, weeklyPeriods?, spontaneousPeriod? }` | Opening hours. `weeklyTimes[]` and `weeklyPeriods[]` each carry `description`/`startDate`/`endDate` plus one entry per weekday (`{ open, hours }`): `weeklyTimes[].<day>.hours` is a flat list of time-of-day strings; `weeklyPeriods[].<day>.hours` is a list of `[start, end]` pairs. Which of the two appears depends on the product/supplier. `spontaneousPeriod` (`{ description, eventDateList }`) covers one-off, non-weekly schedules. Empty on most products. |
| `redemptionMethod` | string | E.g. `DIGITAL`. |
| `handiAccess` | boolean | Accessibility flag. |
| `supplier` | `{ id, name, externalId, creationDate, updateDate?, contact }` | The product's supplying partner - not the booking engine. |
| `creationDate` / `updateDate` | string (ISO 8601 with offset) | Catalogue record timestamps, not activity-availability dates. |
| `discountPercentage` | float | Present only on discounted products. |
| `relevanceScore` | double | Recommendation score. |
| `scores` | `Scores` | `{ relevanceScore, details: ScoreDetails }` - a breakdown of `relevanceScore`. |
| `aiGenerated` | `{ translation?: boolean }` | Flags which parts of the product are machine-generated, e.g. `{ "translation": true }` when `description`/`title` are a machine translation. |

### `Category`

| Field | Type |
|---|---|
| `level` | int32 |
| `code` | string |
| `name` | string |
| `description` | string |

### `Taxonomy` node

| Field | Type | Notes |
|---|---|---|
| `uuid` | string | |
| `name` | string | |
| `level` | int32 | |
| `slug` | string | E.g. `activities-culture`. |
| `family` | string | E.g. `activities`. |
| `parentUuid` | string | |
| `nameFullPath` | string | Full breadcrumb, e.g. `"Culture#Tickets & passes"`. |
| `nameTranslation` | string | `name` translated into the response's `displayedLanguage`. |
| `startDate` | string (ISO 8601) | When this taxonomy node took effect. |
| `value` | double | Confidence/weight. |
| `version` | int32 | |
| `isCurrent` | boolean | |

`taxonomies` is `Taxonomy[][]` - each inner array represents a single path through the tree, ordered root → leaf.

## Errors

| HTTP | Meaning |
|---|---|
| 400 | Bad request - payload validation. |
| 401 | Unauthorized - a missing, malformed or unrecognized `X-API-Key`. |
| 403 | Forbidden. |
| 404 | No products found (top-activities), or the requested `productId` doesn't exist (get product). |
| 429 | Too many requests - back off and retry with exponential backoff. |
| 500 | Internal server error. |

## Behaviour notes

- **Response shape.** Top-activities and recommended products each return a `GetProductResponse[]` directly; get product returns a single `GetProductResponse`.
- **`id` is not stable, `productId` is.** `id` encodes the context of the search that returned the product, so the same product comes back with a different `id` on each call. Persist and compare on `productId`; if it is absent, reject or omit the product instead of substituting another field. Both are int64; `id` may be large enough to round in a 53-bit float, so decode it as a 64-bit integer or a string. Get product is the exception: since there's no search context, `id` comes back equal to `productId` there - that doesn't make `id` a substitute for `productId` on the search endpoints.
- **Deprecated fields.** `price`, `priceRange`, and the singular `countryCode` on filters - prefer `minPrice`/`maxPrice` and `countryCodes` (with `include`/`exclude`).
- **HTML in `description`.** Sanitize before injecting into the DOM.
- **Booking handoff.** `bookingUrl` is the integration point - redirect the user to the booking engine.
- **Pagination.** Not provided on these endpoints - use `maxRecommendations` to cap results.
- **Multiple location params.** When multiple location params are provided, the API applies priority: coordinates (+ optional `radius`) → `iataCode` → city name. No error is thrown. **The two endpoints spell the locators differently**: the GET takes `lat`+`lon` and `cityName`, the POST body takes `geoLocation` and `city`. Passing the POST names as query parameters returns `400 Invalid param`.
- **`interests` array length.** Recommendation quality degrades as the `interests` list grows. Prefer 1–3 focused items over a long list.
- **`categories` has no discovery path.** There's no taxonomy endpoint, so a valid code is only knowable by having already seen it on a product's `categories`/`taxonomies` in a prior response. Not every product carries a taxonomy, so filtering on `categories` silently excludes untagged-but-relevant products rather than erroring - treat it as an opt-in precision filter once you have confirmed codes, not a default way to narrow results. `interests` has no such coverage gap.
