# Nearest airports - API reference

Detailed reference for `GET /v1/reference-data/locations/airports`. The base URL and authentication model (`X-API-Key` header from `AMADEUS_API_KEY`) are documented in [`../SKILL.md`](../SKILL.md#gateway-and-authentication).

## Endpoint

```
GET https://api.amadeus.com/v1/reference-data/locations/airports
```

Returns the most relevant airports within a radius of a given point.

## Query parameters

| Parameter | Required | Type | Default | Notes |
|---|---|---|---|---|
| `latitude` | yes | number | - | Decimal degrees. |
| `longitude` | yes | number | - | Decimal degrees. |
| `radius` | no | integer | `500` | Search radius in km (no other unit is supported). Range `0`–`500`. |
| `sort` | no | enum | `relevance` | See sort modes below. |
| `page[limit]` | no | integer | `10` | URL-encode brackets where required. |
| `page[offset]` | no | integer | `0` | |

### `sort` modes

| Value | Description |
|---|---|
| `relevance` *(default)* | Score combining distance and traffic - best general-purpose default. |
| `distance` | Purely by straight-line distance from the input point. |
| `analytics.flights.score` | Approximate flight-volume score (last reference year). |
| `analytics.travelers.score` | Approximate traveler-volume score (last reference year). |

## Successful response (200)

```json
{
  "meta": {
    "count": 28,
    "links": {
      "self": "...",
      "next": "...",
      "last": "..."
    }
  },
  "data": [ /* Location[] (subType always AIRPORT) */ ]
}
```

### `Location` schema (airport-only variant)

| Field | Type | Notes |
|---|---|---|
| `type` | `"location"` | Constant. |
| `subType` | `"AIRPORT"` | Always - this endpoint only returns airports. |
| `name` | string | Short name (e.g. `HEATHROW`). |
| `detailedName` | string | E.g. `LONDON/GB:HEATHROW`. |
| `timeZoneOffset` | string | E.g. `+01:00`. |
| `iataCode` | string | 3-letter IATA. |
| `geoCode` | `{ latitude, longitude }` | Numbers. |
| `address` | object | See [airport-city-search reference](../../airport-city-search/references/api-reference.md#address-schema). |
| `distance` | `{ value: integer, unit: "KM" }` | Straight-line distance from the input point. `unit` is always `"KM"` - no alternate unit is supported. |
| `analytics` | `{ flights: { score }, travelers: { score } }` | Both scores returned for this endpoint. |
| `relevance` | number | Always present, regardless of `sort` - it's a property of the airport, not an artifact of the sort mode used to order the page. |

## Errors

| HTTP | Code | Title |
|---|---|---|
| 400 | 477 | INVALID FORMAT |
| 400 | 572 | INVALID OPTION |
| 400 | 4926 | INVALID DATA RECEIVED |
| 400 | 32171 | MANDATORY DATA MISSING |
| 429 | - | TOO MANY REQUESTS |
| 500 | 141 | SYSTEM ERROR HAS OCCURRED |

A 429 means the gateway's rate limit for your API key was hit - back off and retry with exponential backoff.

Example 400:

```json
{
  "errors": [
    {
      "status": 400,
      "code": 32171,
      "title": "MANDATORY DATA MISSING",
      "detail": "missing required parameter",
      "source": { "parameter": "latitude" }
    }
  ]
}
```

## Behaviour notes

- **Civilian airports only** with at least several commercial flights per week. Tiny strips and military fields are excluded.
- **Distances are straight-line, not driving distance.** They don't follow roads, so the real driving distance will usually be longer. Present them as "X km away", not "X km drive" or a drive-time estimate.
- **`relevance` is always present**, regardless of `sort` - `sort` only changes result order, not which fields come back.
- **`subType` is always `AIRPORT`** for this endpoint. If you need cities or a unified picker, use [airport-city-search](../../airport-city-search/SKILL.md).
- **Pagination caveat.** Total count comes back via `meta.count`; `meta.links` carries `self`/`next`/`last`/`first`/`previous` URLs ready to follow.
