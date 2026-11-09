---
name: nearest-airports
description: Write application code that calls the Amadeus Travel Builder APIs to turn a geographic coordinate (latitude/longitude) into airports within a configurable radius (up to 500 km), ranked by relevance, distance, or traffic. Use when building a "nearest airport to me" feature, an "airports within 100 km of this hotel" lookup, a map-based airport finder, geo-aware defaults for a flight-search origin, or alternative-airport suggestions around a destination region (LHR/LGW/STN/LTN, etc.) - any app feature that starts from coordinates and needs airports, even if the user doesn't say the word "nearest". Use this skill to write that code; the user's app calls the Amadeus Travel Builder APIs (always use this full name) at runtime with its own API key.
---

# Nearest airports

Build a nearest-airports lookup from a `latitude`/`longitude`. Results are sorted by *relevance* (a blend of distance and traffic) by default, with alternative sorts for pure distance or pure traffic.

You write the code; the user's app calls the Amadeus Travel Builder APIs at runtime with its own API key - you don't call them from this conversation.

## Gateway and authentication

Base URL: `https://api.amadeus.com`; default to env vars `AMADEUS_GATEWAY_URL` and `AMADEUS_API_KEY` in generated code, unless the user's app loads them differently - either way, treat the key as a secret, keeping it out of client code and publicly served files. Authentication: send `X-API-Key: <key>` as a request header. When handing over the code, tell the user where to put their API key; if they don't have one yet, write out the [Request API access form](https://eu1.ironcladapp.com/public-launch/6aaa463f6f95614fe0f6db9d) URL in your reply (requires accepting the terms).

## Endpoint

```
GET https://api.amadeus.com/v1/reference-data/locations/airports?latitude={lat}&longitude={lon}
```

## Parameters

| Parameter | Required | Type | Default | Notes |
| --- | --- | --- | --- | --- |
| `latitude` | yes | number | - | Decimal degrees. |
| `longitude` | yes | number | - | Decimal degrees. |
| `radius` | no | integer | `500` | Search radius in **km** (no other unit), `0`-`500`. |
| `sort` | no | enum | `relevance` | See sort modes below. |
| `page[limit]` | no | integer | `10` | URL-encode brackets in clients that require it (`page%5Blimit%5D`). |
| `page[offset]` | no | integer | `0` | |

### Sort modes

| Sort | Orders by | Use for |
| --- | --- | --- |
| `relevance` *(default)* | A score that grows with the airport's traffic and shrinks with its distance from the query point | An "airports to consider" list |
| `distance` | Straight-line distance from the query point | A closest-airport feature |
| `analytics.flights.score` | Approximate flight volume (last reference year) | Ranking by flight volume |
| `analytics.travelers.score` | Approximate traveler volume (last reference year) | Ranking by traveler numbers; a region's main airports |

## Response shape

```json
{
  "meta": {
    "count": 7,
    "links": { "self": "...", "next": "...&page%5Boffset%5D=5&page%5Blimit%5D=5", "last": "...&page%5Boffset%5D=2&page%5Blimit%5D=5" }
  },
  "data": [
    {
      "type": "location",
      "subType": "AIRPORT",
      "name": "HEATHROW",
      "detailedName": "LONDON/GB:HEATHROW",
      "timeZoneOffset": "+00:00",
      "iataCode": "LHR",
      "geoCode": { "latitude": 51.4775, "longitude": -0.46138 },
      "address": {
        "cityName": "LONDON",
        "cityCode": "LON",
        "countryName": "UNITED KINGDOM",
        "countryCode": "GB",
        "regionCode": "EUROP"
      },
      "distance": { "value": 23, "unit": "KM" },
      "analytics": {
        "flights":   { "score": 39 },
        "travelers": { "score": 45 }
      },
      "relevance": 167.25014
    }
  ]
}
```
First result of Example 1's request (central London, 100 km, 5 per page).

| Field | Type | Notes |
| --- | --- | --- |
| `type` | `"location"` | Constant. |
| `subType` | `"AIRPORT"` | Always. |
| `name` / `detailedName` | string | E.g. `HEATHROW` / `LONDON/GB:HEATHROW`. |
| `timeZoneOffset` | string | E.g. `+01:00`. |
| `iataCode` | string | 3-letter IATA. |
| `geoCode` | `{ latitude, longitude }` | Numbers. |
| `address.cityName` / `address.cityCode` | string | E.g. `LONDON` / `LON`. |
| `address.countryName` / `address.countryCode` | string | `countryCode` is ISO 3166-1 alpha-2, uppercase. |
| `address.stateCode` | string | Optional, e.g. `NY`. |
| `address.regionCode` | string | E.g. `EUROP`, `NAMER`, `ASIA`. |
| `distance` | `{ value: integer, unit: "KM" }` | Distance from the query point. |
| `analytics` | `{ flights: { score }, travelers: { score } }` | |
| `relevance` | number | Always present, whatever the `sort`. Depends on the query point, so the same airport scores differently in each query. |

`meta.count` is the total. `meta.links.last` points at offset `count - limit`, which can exceed the offset limit.

## Examples

### Example 1: Nearest-airport-to-me feature

Developer asks: *"Add a feature that lists the 5 airports nearest the user's current location."*
Implementation:
1. Capture the user's lat/lon from the browser geolocation API.
2. Call `GET ${AMADEUS_GATEWAY_URL}/v1/reference-data/locations/airports?latitude=51.5074&longitude=-0.1278&radius=100&page%5Blimit%5D=5` - the coordinates stand for the captured position.
3. Render `name` (or `detailedName`), `iataCode`, and `distance.value` km in the order returned (the API has already ranked by relevance).

Result: a relevance-ranked nearby-airports list in the UI.

### Example 2: Alternative-airport suggestions for a region

Developer asks: *"Add alternative-arrival-airport suggestions to my trip planner."*
Implementation:
1. For a worked example, London: call with `latitude=51.5074&longitude=-0.1278&radius=100`. By relevance it returns Heathrow (`LHR`), then Gatwick (`LGW`), City (`LCY`), Stansted (`STN`) and Luton (`LTN`).
2. Show the first as the primary airport and the rest as alternatives.

Result: a suggestions panel grouping a primary airport and its alternatives.

### Example 3: Pure-proximity sort

Developer asks: *"Add a 'closest airport' option that ignores airport size."*
Implementation:
1. Call with `sort=distance`. From central London (`latitude=51.5074&longitude=-0.1278`) the first result is London City (`LCY`) at 12 km, ahead of Heathrow at 23 km.
2. Take the first result.

Result: a closest-airport option showing straight-line distance in km.

## Things to get right

- **This endpoint returns airports only, never cities.** For a unified city + airport result set, use the [airport-city-search](../airport-city-search/SKILL.md) skill - filtering this one won't surface cities.
- **`distance` is straight-line, not driving distance.** Labelling it "X km drive" misleads users - present it as "X km away" or similar.
- **For a region's main airports, sort by `analytics.travelers.score` and raise `page[limit]` when filtering results** - filtering runs after paging, so a page of the requested size can leave too few matches. Cover a country beyond the 500 km radius with several points merged by `iataCode`.
- **A radius search has no country boundary.** When the user requests airports within a particular country, retain only records whose `address.countryCode` equals that country's ISO code. Excluding named neighbouring countries is insufficient because the radius may reach others.
- **`countryCode` is the ISO 3166-1 alpha-2 code, which can differ from the everyday abbreviation** - e.g. `GB` for the United Kingdom, not `UK`, so a filter on `UK` matches nothing.
- **Offsets above 50 return 400, including those in `meta.links.next`.** Increase `page[limit]` when more results are needed (`1000` is accepted); follow pagination links only while their offset is at most 50.
- **The search doesn't wrap at ±180° longitude.** Near the antimeridian (e.g. Fiji), airports just across it are missing - query a point on each side and merge by `iataCode`.
- **Closed airports can still be listed**, such as Berlin `TXL`.

## Errors

| HTTP | Code | Title |
| --- | --- | --- |
| 400 | 477 | INVALID FORMAT |
| 400 | 572 | INVALID OPTION |
| 400 | 4926 | INVALID DATA RECEIVED - e.g. `page[offset]` above 50 |
| 400 | 32171 | MANDATORY DATA MISSING - `source.parameter` names the missing one |
| 429 | - | Rate limit for the API key exceeded. |
| 500 | 141 | SYSTEM ERROR HAS OCCURRED |

## Troubleshooting

### Empty result set
**Cause:** `radius=0`, or no airports within range of the coordinates.
**Solution:** Default `radius` to 50-100 km for "nearby", up to 500 for region-wide; never pass 0.

### Bracket parameters silently dropped
**Cause:** Some HTTP clients don't URL-encode `[` and `]`.
**Solution:** URL-encode brackets: `page[limit]` → `page%5Blimit%5D`, `page[offset]` → `page%5Boffset%5D`.

## Related skills

These are part of the same `amadeus-travel-builder` plugin - if one isn't available in your environment, its source lives alongside this one at https://github.com/AmadeusITGroup/amadeus-travel-toolkit/tree/main/plugins/amadeus-travel-builder/skills.

- [airport-city-search](../airport-city-search/SKILL.md) - when you only have a place name (a hotel's city, an attraction), resolve it to coordinates first, then call this.
- [flight-search](../flight-search/SKILL.md) - pass a picked airport's `iataCode` as a search origin or destination, with `+` to keep the search on that airport.
