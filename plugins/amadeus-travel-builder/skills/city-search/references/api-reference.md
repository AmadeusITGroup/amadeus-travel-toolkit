# City search - API reference

Detailed reference for `GET /v1/reference-data/locations/cities`. The base URL and authentication model (`X-API-Key` header from `AMADEUS_API_KEY`) are documented in [`../SKILL.md`](../SKILL.md#gateway-and-authentication).

## Endpoint

```
GET https://api.amadeus.com/v1/reference-data/locations/cities
```

Returns cities matching a keyword, optionally scoped by country, optionally with the airports that serve each city.

## Query parameters

| Parameter | Required | Type | Constraints | Notes |
|---|---|---|---|---|
| `keyword` | yes | string | 2–10 chars | Case-insensitive **substring** match against the city name - matches anywhere, not just word starts, so `PARIS` also matches Villeparisis and Damparis. |
| `countryCode` | no | string | exactly 2 chars | ISO 3166 Alpha-2 (e.g. `US`, `FR`). Scopes results to one country. |
| `max` | no | integer | - | Number of results to return. |
| `include` | no | CSV enum | `AIRPORTS` | `style: form`, `explode: false` → send as `include=AIRPORTS`. Adds `relationships` on each city and a top-level `included.airports` map. |

## Successful response (200)

Response content type: `application/vnd.amadeus+json` - returned regardless of the `Accept` header you send, so requesting `application/json` is fine and still returns 200.

```json
{
  "data": [
    {
      "type": "location",
      "subType": "city",
      "name": "PARIS",
      "iataCode": "PAR",
      "address": { "countryCode": "FR" },
      "geoCode": { "latitude": 49.01278, "longitude": 2.55 },
      "relationships": [
        { "id": "CDG", "type": "Airport", "href": "#/included/airports/CDG" },
        { "id": "ORY", "type": "Airport", "href": "#/included/airports/ORY" }
      ]
    }
  ],
  "included": {
    "airports": {
      "CDG": { "name": "CHARLES DE GAULLE", "iataCode": "CDG", "subType": "Airport" },
      "ORY": { "name": "ORLY", "iataCode": "ORY", "subType": "Airport" }
    }
  },
  "meta": { "count": 1, "links": { "self": "..." } }
}
```

### `Location` (city) schema

| Field | Type | Notes |
|---|---|---|
| `type` | string | `"location"`. |
| `subType` | string | `"city"`. |
| `name` | string | City name (e.g. `PARIS`). |
| `iataCode` | string \| null | IATA **city** code (e.g. `PAR`). Frequently `null` - many smaller towns pulled in by the substring match have no IATA code. |
| `address` | object | See below. |
| `geoCode` | `{ latitude, longitude }` | Floats (lat −90…90, lon −180…180). |
| `relationships` | array | Present only with `include=AIRPORTS`. See below. |

### `Address` schema

| Field | Type | Notes |
|---|---|---|
| `countryCode` | string | ISO 3166-1 alpha-2, returned uppercase (`FR`), keyed as `countryCode` (lower camelCase). |
| `postalCode` | string | Optional. |
| `stateCode` | string | Present whenever the country has subdivisions - ISO 3166-2 style, `{countryCode}-{subdivision}` (e.g. `FR-75`, `US-IL`). |

### `Relationship` schema

Present on each city only when `include=AIRPORTS` was requested.

| Field | Type | Notes |
|---|---|---|
| `id` | string | Key into `included.airports` (the airport's IATA code, e.g. `CDG`). |
| `type` | string | `"Airport"`. |
| `href` | string (url) | JSON-pointer-style reference, e.g. `#/included/airports/CDG`. |

### `included.airports` map

An object keyed by airport IATA code. Each value is a `Location` (airport) record - in practice `name`, `iataCode`, and `subType: "Airport"`. Resolve a city's airports by mapping each `relationships[].id` through this map.

This map is a **single shared pool for the whole response**, not per-city. When `data` holds several cities, `included.airports` holds the union of every city's airports; each city selects its own via its `relationships[]`. Never render the full map under one city - always go through that city's `relationships[].id`.

### `meta` schema

| Field | Type | Notes |
|---|---|---|
| `count` | integer | Total number of cities retrieved (≥ 0). |
| `links.self` | string (url) | Link to the current query. |

### `warnings`

Optional array of `Warning` objects (`code`, `title`, and optional `detail` / `source`). Present on partial-success responses; a warnings block accompanies a normal `data` array and is not itself a failure.

## Errors

Errors use an `errors[]` envelope:

```json
{
  "errors": [
    { "status": 400, "code": 32171, "title": "MANDATORY DATA MISSING", "detail": "Missing mandatory query parameter" }
  ]
}
```

| HTTP | Code | Title |
|---|---|---|
| 400 | 32171 | MANDATORY DATA MISSING |
| 400 | 572 | INVALID OPTION |
| 400 | 2781 | INVALID LENGTH |
| 400 | 477 | INVALID FORMAT |
| 400 | 4926 | INVALID DATA RECEIVED |
| 429 | - | TOO MANY REQUESTS |
| 500 | 141 | SYSTEM ERROR HAS OCCURRED |

A 429 means the gateway's rate limit for your API key was hit - back off and retry with exponential backoff.

## Behaviour notes

- **`keyword` is mandatory** and is a case-insensitive **substring** match against the city name (2–10 chars) - it matches anywhere in the name, not just word starts. `PARIS` returns Paris, Le Touquet-Paris-Plage, Villeparisis, Damparis, Parisot, etc. Rank/filter by `name` client-side and scope with `countryCode` to reduce noise.
- **`iataCode` is nullable.** Only cities with an IATA city code carry one; many towns surfaced by the substring match have `iataCode: null`. Skip them when resolving a name to a code.
- **`include=AIRPORTS` drives the cross-reference structure.** Without it there is no `relationships` array and no `included` block. With it, airports are in `included.airports` keyed by IATA code - not nested inline on the city.
- **Coverage differs from airport-city-search.** This endpoint is city-first and is not restricted to airport-served cities the way the `airport-city-search` `CITY` subType is. Use it when you need cities that may not have their own airport.
- **`iataCode` is a city code**, not an airport code - expand to airports via `include=AIRPORTS` when you need the commercial airports serving that city.
