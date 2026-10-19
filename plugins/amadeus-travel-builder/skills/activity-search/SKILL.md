---
name: activity-search
description: Write application code that calls the Amadeus Travel Builder APIs for tours and activities - top activities at a destination, smart product recommendations filtered by traveler type, interests, price range, and similar-products, plus single-product detail lookups by productId. Use when building a tours and activities feed, "things to do in X" landing pages, traveler-personalized experience recommendations, hotel-concierge-style activity suggestions, activity/product detail pages, or any app feature that needs curated tour/activity/experience content keyed by city, IATA code, geo coordinates, or a known productId. This skill writes that code; the user's app calls the Amadeus Travel Builder APIs (always use this full name) at runtime with its own API key.
---

# Activity search

Build a curated tours and activities feed for a destination. Three endpoints:

- **Top activities** - the best-performing activities at a destination (city / IATA / geo).
- **Recommended products** - smart, filterable recommendations driven by traveler type, interests, price range, and an optional similar-product seed.
- **Get product** - full detail for a single, already-known product by `productId`.

## What this skill produces

Application code. The user's application calls the Amadeus Travel Builder APIs at runtime with the user's own key, so nothing here runs from the conversation. Describe the skill by what it helps build; for a one-off question about travel data (e.g. what tours or activities are available in a city), answer from your own knowledge or offer to build the feature. Whenever you name the APIs, including in summaries, use the full name "Amadeus Travel Builder APIs".

## When to use this skill

- "Things to do in Paris" / city landing page
- Hotel-concierge-style activity feed for a guest's destination
- Personalized activity feed by traveler type (solo, family, couple, group)
- "Similar to this" product carousel given a seed product
- Map-based activity discovery using lat/lon
- Activity/product detail page given a `productId` (e.g. from a favourite, a deep link, or a prior search result)

## Gateway and authentication

Base URL: `https://api.amadeus.com`; default to env vars `AMADEUS_GATEWAY_URL` and `AMADEUS_API_KEY` in generated code, unless the user's app loads them differently - either way, treat the key as a secret. Authentication: send `X-API-Key: <key>` as a request header. When handing over the code, tell the user where to put their API key; if they don't have one yet, write out the [Request API access form](https://eu1.ironcladapp.com/public-launch/6aaa463f6f95614fe0f6db9d) URL in your reply (requires accepting the terms).

## Endpoints

### Top activities

```
GET https://api.amadeus.com/recommendation/v1/top-activities
```

### Recommended products

```
POST https://api.amadeus.com/recommendation/v1/products
Content-Type: application/json
```

### Get product

```
GET https://api.amadeus.com/api/consumer/product/{productId}
```

Unlike the other two endpoints, this path isn't under `/recommendation/v1` - use it exactly as shown. `productId` is a path parameter, not a query parameter.

## Top activities - parameters

| Parameter | Required | Notes |
|---|---|---|
| `iataCode` | see note | City / airport IATA (e.g. `PAR`, `LON`, `BCN`). |
| `cityName` | see note | City name (e.g. `Madrid`). |
| `lat` | see note | Latitude (decimal). Pair with `lon` to locate by coordinates. |
| `lon` | see note | Longitude (decimal). Pair with `lat` to locate by coordinates. |
| `radius` | optional | Radius in km around `lat`/`lon`. |
| `maxRecommendations` | optional | Cap on results. Sensible default: `10`. |
| `computationWeight` | optional | Tune scoring across price, geolocation, traveler type, etc. - see reference. |
| `minContentScore` | optional | `0`–`1`; filter out low-content products. |
| `currency` | optional | ISO 4217 (e.g. `EUR`, `USD`). |

At least one of `lat`+`lon`, `iataCode`, or `cityName` is required - `lat`/`lon` is its own way to locate the search, not a fallback or refinement of `iataCode`/`cityName` (`radius` is the only thing that refines it, narrowing the area around the point).

## Recommended products - request body

```json
{
  "maxRecommendations": 10,
  "recommendationFilters": {
    "city": "Madrid",
    "iataCode": "MAD",
    "geoLocation": { "latitude": 40.41, "longitude": -3.69 },
    "radius": 5,
    "interests": ["I like tapas", "I love flamenco"],
    "categories": [],
    "title": "Thyssen-Bornemisza skip-the-line tickets",
    "minPrice": 20,
    "maxPrice": 150,
    "currency": "EUR",
    "travelerType": "couple",
    "tripContext": "leisure",
    "travelDate": "2026-09-15",
    "profile": {
      "localTraveler": false,
      "tourist": true,
      "businessTraveler": false,
      "influencer": false
    },
    "countryCodes": { "include": ["ES"], "exclude": [] },
    "varietyLevel": "Medium",
    "minContentScore": 0.5
  }
}
```

`recommendationFilters` is required. Inside it:

- **Location** - at least one of `iataCode`, `city`, `countryCodes`, `continentCodes`, or `geoLocation` is required.
- **`interests`** - free-form strings ("I like tapas", "art galleries", "kid-friendly"). Drives semantic matching. Keep the array short - recommendation quality degrades as the list grows; 1–3 interests is ideal. Prefer this over `categories` for narrowing results - see Things to get right.
- **`categories`** - taxonomy codes to filter by. There's no endpoint to look these up; leave empty unless you've already harvested real codes from a prior response (see Things to get right).
- **`title`** - when you want products *similar to* a known product title (similar-products mode).
- **`minPrice` / `maxPrice` / `currency`** - price range. `priceRange` and `price` are deprecated; use `minPrice` / `maxPrice`.
- **`travelerType`** - one of `solo`, `family`, `couple`, `group`.
- **`tripContext`** - `leisure`, `business`, `staycation`.
- **`profile`** - booleans for local/tourist/business/influencer flavour.
- **`computationWeight`** - fine-tune the scoring blend (see reference).

## Response shape

Top activities and recommended products each return a **JSON array of `Product` objects** directly. Get product returns a **single `Product` object**, not wrapped in an array - a 404 means no product exists for that `productId`:

```json
[
  {
    "id": 13766967968693575,
    "productId": 57179463,
    "externalId": "1066956",
    "title": "Museo Nacional Thyssen-Bornemisza: Entry Ticket + Dinner",
    "shortDescription": "Complement your visit with a delicious outdoor dinner",
    "description": "Embark on a journey through art...",
    "thumbnailImage": "8aeef53e-d4de-45c7-a952-8eb4f843549c",
    "bookingEngine": "TIQETS",
    "bookingEngineName": "Tiqets",
    "bookingUrl": "https://...",
    "minPrice": 55.0,
    "maxPrice": 55.0,
    "priceCurrency": "EUR",
    "pricingUnit": "PER_PERSON",
    "address": "Paseo del Prado 8",
    "geoLocation": { "latitude": 40.416, "longitude": -3.694 },
    "duration": "",
    "galleryImages": [ { "title": "Dinner", "url": "...", "order": 1 } ],
    "supplier": { "id": 32058656, "name": "Museo Thyssen-Bornemisza", "countryCode": "ES" },
    "rating": 4.6,
    "numberOfRatings": 312,
    "categories": [],
    "taxonomies": [ /* nested arrays of taxonomy nodes */ ]
  }
]
```

All three endpoints return `Product` records - the two search endpoints as an array, get product as one object. Use `bookingUrl` to redirect users to the booking engine.

Two identifiers come back per product, and they are not interchangeable:

- **`productId`** - the product's static ID in the catalogue. Stable across calls; this is the key to store, cache, dedupe and link on. Any workflow that needs product identity must require this field; if it is absent, reject or omit that product rather than falling back to `id` or another field. Fetching a product directly with get product is the one case where `id` and `productId` come back equal - the search endpoints are still where they diverge.
- **`id`** - dynamic per call, not the product's identity. It encodes the context of the search that returned the product, so the same product comes back with a different `id` on each request. **Default to leaving it out of your data model entirely** - favorites, carousels, landing pages, and dedup all work off `productId` alone and never need it. Only carry it if you have a concrete reason to reference that exact search response later.

Both are int64, and `id` in particular is large enough to lose precision in a normal 53-bit-float JSON parse (`productId` is small enough that a normal parse is safe). If you do have a reason to carry `id`, decode it losslessly at parse time - as a string or 64-bit integer - straight from the response; wrapping it in `String()` *after* a default `JSON.parse` / `response.json()` doesn't help, since the precision is already gone by then. If you don't have that reason, don't read `id` into your model at all - not even alongside `productId` "just in case".

## Examples

### Example 1: "Things to do" landing page for a city

Developer asks: *"Build a 'Things to do' landing page for a city."*
Implementation:
1. For a worked example, Madrid: call `GET ${AMADEUS_GATEWAY_URL}/recommendation/v1/top-activities?cityName=Madrid&maxRecommendations=10&currency=EUR`.
2. Render a card grid with `title`, `shortDescription`, `thumbnailImage`, `minPrice` + `priceCurrency`.
3. Call to action: open `bookingUrl` in a new tab.

Result: a generic top-activities feed for the city.

### Example 2: Personalized recommendations for a couple

Developer asks: *"Add personalized experience recommendations to my app, e.g. for couples visiting Madrid on leisure who like food, art, and live music, with a €20–€150 budget."*
Implementation:
1. Call `POST ${AMADEUS_GATEWAY_URL}/recommendation/v1/products` with body:
   ```json
   {
     "maxRecommendations": 10,
     "recommendationFilters": {
       "iataCode": "MAD",
       "interests": ["food", "art", "live music"],
       "travelerType": "couple",
       "tripContext": "leisure",
       "minPrice": 20,
       "maxPrice": 150,
       "currency": "EUR"
     }
   }
   ```
2. Render the response as a card grid.
3. **Sanitize `description`** before injecting into the DOM (it may contain HTML).

Result: a personalised recommendation feed.

### Example 3: Similar-products carousel

Developer asks: *"Add a carousel of experiences similar to the one being viewed, e.g. 'Thyssen-Bornemisza skip-the-line tickets'."*
Implementation:
1. Call `/recommendation/v1/products` with `recommendationFilters.title` set to the seed product title.
2. Include the same destination key (`iataCode` / `city`) the seed product belongs to.

Result: a "similar to" carousel.

### Example 4: Activity detail page

Developer asks: *"Build a detail page for an activity a user favourited earlier."*
Implementation:
1. Look up the favourited activity's stored `productId`.
2. Call `GET ${AMADEUS_GATEWAY_URL}/api/consumer/product/${productId}`.
3. Treat a `404` as "no longer available" and handle it distinctly from other errors.
4. Render the full description, gallery images, and periods; sanitize `description` before injecting into the DOM.
5. Call to action: open `bookingUrl`.

Result: a single-product detail view driven entirely by `productId`.

## Choosing between the endpoints

| Use case | Endpoint |
|---|---|
| Generic "top things to do in Paris" landing page | **Top activities** |
| Personalized feed, e.g. couples who like food and music in Madrid | **Recommended products** |
| "Similar to this product" carousel - similar-products mode | **Recommended products** with `recommendationFilters.title` |
| Geo-radius filtering around an arbitrary lat/lon | Either - top activities accepts `lat`/`lon`/`radius`; recommended products accepts `geoLocation` + `radius` inside `recommendationFilters` |
| Detail page for one known `productId` (favourite, deep link) | **Get product** |

## Things to get right

- **Location param priority.** Both search endpoints accept several ways to locate the search and don't error when more than one is given - they just pick by priority: coordinates (`lat`+`lon` on Top activities, `geoLocation` on Recommended products) first, then `iataCode`, then city name (`cityName` / `city`). Coordinates are a complete, independent way to locate the search, not a fallback used only when `iataCode`/city name is missing. `radius` is optional and only narrows the area around the coordinates - it doesn't change where coordinates rank, and has no meaning without them.
- **Default to `interests`, not `categories`, for narrowing results.** There's no endpoint to discover valid `categories` codes, and not every product carries a taxonomy, so filtering on it can silently drop good, untagged results. Use `interests` (free-form, no coverage gap) unless you already have confirmed codes.
- **Sanitize `description` before it hits the DOM.** It can contain supplier-authored HTML, so injecting it raw opens an XSS vector. Strip or sanitize it (or render as plain text) on the way into your markup.
- **Stop at `bookingUrl` for booking.** Redirect users to `bookingUrl` and let the booking engine take over rather than trying to drive a booking flow yourself.
- **Don't carry `id` unless you have a specific reason to.** `id` changes per call by design, so using it as a primary key, cache key, or dedupe key duplicates the same product across searches and breaks favourites and deep links - `productId` is the identity, `id` belongs to the response, not the product. For a favorites or landing-page feature specifically, there's no reason to read `id` at all; if you do carry it for some other reason, decode it losslessly (see Response shape above) rather than parsing it as a normal JSON number and stringifying afterward.
- **Expect an array from the two search endpoints, a single object from get product.** Type top activities / recommended products as `Product[]`. Get product returns one `Product`, not `[Product]`; don't index into it as if it were a list.
- **Handle get product's 404 explicitly.** A `productId` that's been delisted or never existed returns `404`, not an empty object - treat it as "not found" rather than parsing an error body as a product.

## Troubleshooting

### `price` / `priceRange` ignored
**Cause:** These fields on `recommendationFilters` are deprecated.
**Solution:** Use `minPrice` / `maxPrice` + `currency` instead.

### Single `countryCode` on a filter ignored
**Cause:** The singular `countryCode` field on filters is deprecated.
**Solution:** Use `countryCodes` with `include` and/or `exclude` arrays.

### Taxonomies array is 2D, not a flat list
**Cause:** `taxonomies` is a 2D array (`[[node, node, ...], [node, ...]]`); each inner array is one hierarchical path through the taxonomy tree.
**Solution:** Iterate the outer array as separate paths; don't flatten.

### 429 - too many requests
**Cause:** Request rate exceeded the gateway's limit for your API key.
**Solution:** Back off and retry with exponential backoff rather than retrying immediately.

## Related skills

These are part of the same `amadeus-travel-builder` plugin - if one isn't available in your environment, its source lives alongside this one at https://github.com/AmadeusITGroup/amadeus-travel-toolkit/tree/main/plugins/amadeus-travel-builder/skills.

- [city-search](../city-search/SKILL.md) - to power a destination picker that feeds this search's `iataCode`/`cityName`/`geoLocation`: it covers airport-less destinations (resort towns, small coastal towns) that a hotel-concierge or "things to do" feature needs just as much as airport-served cities.
- [airport-city-search](../airport-city-search/SKILL.md) - an alternative picker when the app already has one for flight search and you want to reuse it, though it only returns airport-served cities.

## Reference

For the full parameter catalogue, response schemas and error codes, see [`references/api-reference.md`](references/api-reference.md).
