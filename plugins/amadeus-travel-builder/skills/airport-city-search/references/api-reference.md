# Airport & city search - API reference

Detailed reference for `GET /v1/reference-data/locations` and `GET /v1/reference-data/locations/{locationId}`. The base URL and authentication model (`X-API-Key` header from `AMADEUS_API_KEY`) are documented in [`../SKILL.md`](../SKILL.md#gateway-and-authentication).

## Endpoints

```
GET https://api.amadeus.com/v1/reference-data/locations
GET https://api.amadeus.com/v1/reference-data/locations/{locationId}
```

## Keyword search - `GET /v1/reference-data/locations`

### Query parameters

| Parameter | Required | Type | Default | Notes |
|---|---|---|---|---|
| `subType` | yes | CSV of `AIRPORT` / `CITY` | - | At least one value required. |
| `keyword` | yes | string | - | Matches the start of any word in `name` / `detailedName`, or the start of an IATA code. |
| `countryCode` | no | ISO 3166-1 alpha-2 | - | E.g. `US`, `FR`, `GB`. |
| `view` | no | `LIGHT` \| `FULL` | `FULL` | `LIGHT` is for autocomplete; `FULL` includes `geoCode`, `timeZoneOffset`, full `address`, `analytics`. |
| `page[limit]` | no | integer | `10` | URL-encode brackets where required. |
| `page[offset]` | no | integer | `0` | |
| `sort` | no | `analytics.travelers.score` | `analytics.travelers.score` | The only legal value - `sort` exists but doesn't actually support alternatives. |

### Successful response (200)

```json
{
  "meta": {
    "count": 127,
    "links": {
      "self": "...",
      "next": "...",
      "previous": "...",
      "first": "...",
      "last": "..."
    }
  },
  "data": [ /* Location[] */ ]
}
```

### `Location` schema

| Field | Type | Notes |
|---|---|---|
| `type` | `"location"` | Constant. |
| `subType` | `"AIRPORT"` \| `"CITY"` | |
| `name` | string | Short name. |
| `detailedName` | string | Format: `CITY/COUNTRY:NAME` for airports, `CITY/COUNTRY` for cities. |
| `id` | string | Sub-type-prefixed: `A...` (airport) / `C...` (city). Use this with the by-ID endpoint. |
| `self` | `{ href, methods }` | HATEOAS link to the by-ID endpoint. |
| `timeZoneOffset` | string | E.g. `+01:00`. Includes daylight saving. |
| `iataCode` | string | IATA code (3 letters). |
| `geoCode` | `{ latitude, longitude }` | Numbers, though some records arrive as strings - coerce. |
| `address` | object | See below. |
| `analytics` | `{ travelers: { score } }` | Travelers-traffic ranking score. |
| `relevance` | number | Optional - only set in some response paths. |

### `Address` schema

| Field | Type | Notes |
|---|---|---|
| `cityName` | string | Equal to `name` if the location is a city. |
| `cityCode` | string | IATA city code. |
| `countryName` | string | Localized country name. |
| `countryCode` | string | ISO 3166-1 alpha-2, uppercase. Present in `view=FULL` (the default); stripped out in `view=LIGHT`, which carries only `cityName` and `countryName`. |
| `stateCode` | string | Optional (e.g. `DC`, `CA`). |
| `regionCode` | string | E.g. `EUROP`, `NAMER`, `ASIA`. |

### `view=LIGHT` - actual fields returned

The trimmed payload always carries `iataCode`, `name`, `detailedName`, and in practice also:

- `id`
- `self`
- `type`
- `subType`
- `address.cityName`
- `address.countryName`

Type the response from the live shape. `view=FULL` returns the same fields plus `analytics`, `geoCode` and `timeZoneOffset`.

### Errors

| HTTP | Code | Title |
|---|---|---|
| 400 | 477 | INVALID FORMAT |
| 400 | 572 | INVALID OPTION |
| 400 | 2781 | INVALID LENGTH |
| 400 | 4926 | INVALID DATA RECEIVED |
| 400 | 32171 | MANDATORY DATA MISSING |
| 429 | - | TOO MANY REQUESTS |
| 500 | 141 | SYSTEM ERROR HAS OCCURRED |

A 429 means the gateway's rate limit for your API key was hit - back off and retry with exponential backoff.

## By-ID lookup - `GET /v1/reference-data/locations/{locationId}`

### Path parameter

| Parameter | Required | Type | Notes |
|---|---|---|---|
| `locationId` | yes | string | Sub-type-prefixed ID returned from keyword search. `A...` for airports, `C...` for cities. |

### Successful response (200)

Same `Location` shape as keyword search, but `data` is a single object (not an array):

```json
{
  "meta": { "links": { "self": "..." } },
  "data": { "type": "location", "subType": "AIRPORT", "id": "ALHR", ... }
}
```

### Errors

| HTTP | Code | Title |
|---|---|---|
| 400 | 572 | INVALID OPTION |
| 404 | 1797 | NOT FOUND |
| 429 | - | TOO MANY REQUESTS |
| 500 | 141 | SYSTEM ERROR HAS OCCURRED |

A 404 response uses the same `errors[]` envelope as other errors:

```json
{
  "errors": [
    {
      "status": 404,
      "code": 1797,
      "title": "NOT FOUND",
      "detail": "No response found for this query parameter",
      "source": { "parameter": "airport" }
    }
  ]
}
```

## Behaviour notes

- **`subType` is mandatory** on keyword search. The API does not default to "both".
- **Sorting is fixed** at `analytics.travelers.score`. Any other value returns a 400 (`INVALID DATA RECEIVED`) - it does not fall back to the default silently.
- **`countryCode` depends on `view`.** `view=FULL` (the default) returns `countryCode`, `cityCode` and `regionCode` on `address`, plus `stateCode` where applicable; `view=LIGHT` strips `address` down to just `cityName` and `countryName`. It is uppercase ISO 3166-1 alpha-2, same as elsewhere in the referential data (`/locations/airports`, `/locations/cities`).
- **Sub-type-prefixed IDs.** The same IATA code can produce two `Location` records, one per sub-type - `AMAD` (Barajas airport) and `CMAD` (Madrid city) both carry `iataCode: MAD`. Treat `id` as the unique key, not `iataCode`.
