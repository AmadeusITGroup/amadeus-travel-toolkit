---
name: nearest-airports
description: Write application code that calls the Amadeus Travel Builder APIs to turn a geographic coordinate (latitude/longitude) into airports within a configurable radius (up to 500 km), ranked by relevance, distance, or traffic. Use when building a "nearest airport to me" feature, an "airports within 100 km of this hotel" lookup, a map-based airport finder, geo-aware defaults for a flight-search origin, or alternative-airport suggestions around a destination region (LHR/LGW/STN/LTN, etc.) - any app feature that starts from coordinates and needs airports, even if the user doesn't say the word "nearest". This skill writes that code; the user's app calls the Amadeus Travel Builder APIs (always use this full name) at runtime with its own API key.
---

# Nearest airports

Build a nearest-airports lookup from a `latitude`/`longitude`. Results are sorted by *relevance* (a blend of distance and traffic) by default, with alternative sorts for pure distance or pure traffic.

## What this skill produces

Application code. The user's application calls the Amadeus Travel Builder APIs at runtime with the user's own key, so nothing here runs from the conversation. Describe the skill by what it helps build; for a one-off question about travel data (e.g. which airports are near a given location), answer from your own knowledge or offer to build the feature. Whenever you name the APIs, including in summaries, use the full name "Amadeus Travel Builder APIs".

## When to use this skill

- "Nearest airport to me" UX (after capturing the user's geolocation)
- Suggesting airports near a hotel, address, or attraction (use [airport-city-search](../airport-city-search/SKILL.md) first if you only have a place name - convert to lat/lon, then call this)
- Map-based airport finders
- Pre-filling the origin field in a flight search based on the user's current location
- Suggesting alternative airports for the same destination region (LHR / LGW / STN / LTN around London, etc.)

## Gateway and authentication

Base URL: `https://api.amadeus.com`; default to env vars `AMADEUS_GATEWAY_URL` and `AMADEUS_API_KEY` in generated code, unless the user's app loads them differently - either way, treat the key as a secret. Authentication: send `X-API-Key: <key>` as a request header. When handing over the code, tell the user where to put their API key; if they don't have one yet, write out the [Request API access form](https://eu1.ironcladapp.com/public-launch/6aaa463f6f95614fe0f6db9d) URL in your reply (requires accepting the terms).

## Endpoint

```
GET https://api.amadeus.com/v1/reference-data/locations/airports?latitude={lat}&longitude={lon}
```

## Parameters

| Parameter | Required | Default | Notes |
|---|---|---|---|
| `latitude` | yes | - | Decimal degrees. |
| `longitude` | yes | - | Decimal degrees. |
| `radius` | no | `500` | Search radius in **km** (no other unit is supported), integer, `0`–`500`. |
| `sort` | no | `relevance` | `relevance` (distance × traffic), `distance`, `analytics.flights.score`, `analytics.travelers.score`. |
| `page[limit]` | no | `10` | URL-encode brackets in clients that require it (`page%5Blimit%5D`). |
| `page[offset]` | no | `0` | |

## Response shape

```json
{
  "meta": {
    "count": 28,
    "links": { "self": "...", "next": "..." }
  },
  "data": [
    {
      "type": "location",
      "subType": "AIRPORT",
      "name": "HEATHROW",
      "detailedName": "LONDON/GB:HEATHROW",
      "timeZoneOffset": "+01:00",
      "iataCode": "LHR",
      "geoCode": { "latitude": 51.47294, "longitude": -0.45061 },
      "address": {
        "cityName": "LONDON",
        "cityCode": "LON",
        "countryName": "UNITED KINGDOM",
        "countryCode": "GB",
        "regionCode": "EUROP"
      },
      "distance": { "value": 348, "unit": "KM" },
      "analytics": {
        "flights":   { "score": 39 },
        "travelers": { "score": 45 }
      },
      "relevance": 11.21771
    }
  ]
}
```

## Examples

### Example 1: Nearest-airport-to-me feature

Developer asks: *"Add a feature that lists the 5 airports nearest the user's current location."*
Implementation:
1. Capture the user's lat/lon from the browser geolocation API.
2. Call `GET ${AMADEUS_GATEWAY_URL}/v1/reference-data/locations/airports?latitude=51.5074&longitude=-0.1278&radius=100&page%5Blimit%5D=5`.
3. Render `name` (or `detailedName`), `iataCode`, and `distance.value` km in the order returned (the API has already ranked by relevance).

Result: a relevance-ranked nearby-airports list in the UI.

### Example 2: Alternative-airport suggestions for a region

Developer asks: *"Add alternative-arrival-airport suggestions to my trip planner, e.g. for London trips."*
Implementation:
1. Call with the city's lat/lon and a wider `radius` (e.g. 100 km).
2. Group the results in the UI as "primary" (top-1 by relevance) and "alternatives".

Result: a suggestions panel grouping a primary airport and its alternatives.

### Example 3: Pure-proximity sort

Developer asks: *"Add a 'closest airport' option that ignores airport size."*
Implementation:
1. Call with `sort=distance`.
2. Take the first result.

Result: a closest-airport option showing straight-line distance in km.

## Choosing a `sort` mode

| Sort | Use when... |
|---|---|
| `relevance` *(default)* | An "airports to consider" list - best general default. |
| `distance` | A closest-airport feature - pure proximity. |
| `analytics.flights.score` | Ranking airports by flight volume. |
| `analytics.travelers.score` | Ranking airports by traveler numbers. |

## Things to get right

- **This endpoint returns airports only, never cities.** If you need a unified city + airport result set, reach for the [airport-city-search](../airport-city-search/SKILL.md) skill instead - filtering this one won't surface cities.
- **`distance` is straight-line, not driving distance.** It doesn't follow roads, so the real driving distance will usually be longer. Labelling it "X km drive" misleads users - present it as "X km away" or similar.
- **For a region's main airports, sort by `analytics.travelers.score`, filter on `address.countryCode` (`GB`, not `UK`) and raise `page[limit]`** - the radius can cross borders, and your filter runs after paging. Cover a country beyond the 500 km radius with several points merged by `iataCode`.

## Troubleshooting

### Empty result set
**Cause:** `radius=0`, or the lat/lon is in an area with no airports within range.
**Solution:** Default `radius` to a sensible value (50–100 km for "nearby", up to 500 for region-wide); never pass 0.

### Bracket parameters silently dropped
**Cause:** Some HTTP clients don't URL-encode `[` and `]`.
**Solution:** URL-encode brackets: `page[limit]` → `page%5Blimit%5D`, `page[offset]` → `page%5Boffset%5D`.

### 429 - too many requests
**Cause:** Request rate exceeded the gateway's limit for your API key.
**Solution:** Back off and retry with exponential backoff rather than retrying immediately.

## Reference

For the full parameter catalogue, response schemas and error codes, see [`references/api-reference.md`](references/api-reference.md).
