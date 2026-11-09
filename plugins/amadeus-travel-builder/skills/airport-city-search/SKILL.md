---
name: airport-city-search
description: Write application code for a unified airport and city typeahead on the Amadeus Travel Builder APIs - keyword search across airports and cities by name or IATA code, ranked by traveller traffic, plus by-ID lookup. Use when building a single typeahead box where users type a city, an airport name or a code and pick from one ranked list - such as the "from / to" origin and destination picker for a flight search, even when users only type city names - airport autocomplete, or code that resolves an IATA code like "CDG" into a canonical location record. Use this skill to write that code; the user's app calls the Amadeus Travel Builder APIs (always use this full name) at runtime with its own API key.
---

# Airport & City Search

Build keyword search and by-ID lookup over a unified set of airports and cities - the typeahead behind most "where are you flying from / to?" pickers.

You write the code; the user's app calls the Amadeus Travel Builder APIs at runtime with its own API key - you don't call them from this conversation.

## Gateway and authentication

Base URL: `https://api.amadeus.com`; default to env vars `AMADEUS_GATEWAY_URL` and `AMADEUS_API_KEY` in generated code, unless the user's app loads them differently - either way, treat the key as a secret, keeping it out of client code and publicly served files. Authentication: send `X-API-Key: <key>` as a request header. When handing over the code, tell the user where to put their API key; if they don't have one yet, write out the [Request API access form](https://eu1.ironcladapp.com/public-launch/6aaa463f6f95614fe0f6db9d) URL in your reply (requires accepting the terms).

## Endpoints

```
GET https://api.amadeus.com/v1/reference-data/locations?subType={AIRPORT|CITY|AIRPORT,CITY}&keyword={keyword}
GET https://api.amadeus.com/v1/reference-data/locations/{locationId}
```

## Keyword search parameters

| Parameter | Required | Type | Default | Notes |
| --- | --- | --- | --- | --- |
| `subType` | yes | CSV of `AIRPORT` / `CITY` | - | `AIRPORT`, `CITY`, or `AIRPORT,CITY`. No default - omitting it returns 400. |
| `keyword` | yes | string | - | Matches the start of any word in `name` / `detailedName`, or the start of an IATA code. 1-30 characters. |
| `countryCode` | no | ISO 3166-1 alpha-2 | - | E.g. `US`, `FR`; narrows a keyword search by country. |
| `view` | no | `LIGHT` \| `FULL` | `FULL` | `LIGHT` for a label-only autocomplete (smaller payload). `FULL` when the pick feeds a map, groups airports under their city, or is passed to another call. |
| `page[limit]` | no | integer | `10` | URL-encode brackets in clients that require it (`page%5Blimit%5D`). |
| `page[offset]` | no | integer | `0` | |
| `sort` | no | string | `analytics.travelers.score` | The only accepted value; omit it. |

## By-ID lookup parameter

| Parameter | Required | Notes |
| --- | --- | --- |
| `locationId` *(path)* | yes | The record's `id` from keyword search - `A...` for airports (`ALHR` = Heathrow), `C...` for cities (`CPAR` = Paris). Case-sensitive. The by-ID lookup takes no query parameters, not even `view`. |

## Response shape

Keyword search:

```json
{
  "meta": {
    "count": 98,
    "links": { "self": "...?subType=AIRPORT,CITY&keyword=LON&page%5Blimit%5D=2", "next": "...&page%5Boffset%5D=2&page%5Blimit%5D=2", "last": "...&page%5Boffset%5D=96&page%5Blimit%5D=2" }
  },
  "data": [
    { "type": "location", "subType": "CITY", "name": "LONDON", "id": "CLON", "iataCode": "LON", "...": "..." },
    {
      "type": "location",
      "subType": "AIRPORT",
      "name": "HEATHROW",
      "detailedName": "LONDON/GB:HEATHROW",
      "id": "ALHR",
      "self": { "href": "https://api.amadeus.com/v1/reference-data/locations/ALHR", "methods": ["GET"] },
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
      "analytics": { "travelers": { "score": 45 } }
    }
  ]
}
```
The response to `subType=AIRPORT,CITY&keyword=LON&page[limit]=2`, the London city record trimmed.

By-ID lookup returns the same record as a single object under `data`, not an array.

| Field | Type | Notes |
| --- | --- | --- |
| `type` | `"location"` | Constant. |
| `subType` | `"AIRPORT"` \| `"CITY"` | |
| `name` | string | Short name. |
| `detailedName` | string | Display label: `CITY/COUNTRY:NAME` for airports (`LONDON/GB:HEATHROW`), `CITY/COUNTRY` for cities. Some countries add a state (`LONGREACH/QL/AU`), and an airport named after its city has no `:NAME`. |
| `id` | string | Sub-type-prefixed: `A...` (airport) / `C...` (city). |
| `self` | `{ href, methods }` | Link to the by-ID endpoint. |
| `timeZoneOffset` | string | E.g. `+00:00`; daylight saving included, so LHR is `+01:00` in summer. `FULL` only. |
| `iataCode` | string | 3 letters. |
| `geoCode` | `{ latitude, longitude }` | Numbers. `FULL` only. |
| `address.cityName` | string | Equal to `name` for a city. |
| `address.cityCode` | string | IATA city code. `FULL` only. |
| `address.countryName` | string | |
| `address.countryCode` | string | ISO 3166-1 alpha-2, uppercase. `FULL` only. |
| `address.stateCode` | string | Optional, e.g. `DC`. `FULL` only. |
| `address.regionCode` | string | E.g. `EUROP`, `NAMER`. `FULL` only. |
| `analytics` | `{ travelers: { score } }` | Optional - missing on many small airports and cities. `FULL` only. |

`view=LIGHT` returns `id`, `self`, `type`, `subType`, `iataCode`, `name`, `detailedName` and `address` with only `cityName` and `countryName` - type against that shape.

## Examples

### Example 1: From / to airport typeahead

Developer asks: *"Add an airport picker to my flight form."*
Implementation:
1. Transliterate the typed text to ASCII and percent-encode it on its own (spaces as `%20`, not `+`). For a worked example, the user types `Paris`: call `GET ${AMADEUS_GATEWAY_URL}/v1/reference-data/locations?subType=AIRPORT,CITY&keyword=Paris&view=LIGHT&page%5Blimit%5D=8`, which returns Paris (`CPAR`) followed by Charles de Gaulle (`ACDG`), Orly (`AORY`) and Beauvais (`ABVA`).
2. Render `name` (or `detailedName`) plus `iataCode`, with a `subType` badge (`AIRPORT` vs `CITY`).
3. Store the selected record - its `id` (sub-type-prefixed, e.g. `ALHR`), `iataCode` and `subType` - in your form state, not just the label shown in the input.

Result: a typeahead that resolves user text into IATA codes and canonical location records.

### Example 2: Resolve a stored selection back to its full record

Developer asks: *"Add a recent-airport view that reloads the full record for the airport a user picked last time."*
Implementation:
1. Read the stored `id` (e.g. `ALHR`).
2. Call `GET ${AMADEUS_GATEWAY_URL}/v1/reference-data/locations/ALHR`.
3. Render the unwrapped `data` object; show "not found" on a `404` rather than a "bad request" error.

Result: code that restores a stored selection to its full record.

### Example 3: Country-scoped city picker

Developer asks: *"Let users search for German cities only."*
Implementation:
1. For a worked example, the user types `MUN`: call with `subType=CITY&countryCode=DE&keyword=MUN`, which returns Munich (`MUC`).
2. Render the returned cities.

Result: typeahead constrained to German cities.

## Things to get right

- **Send `keyword` as ASCII, with spaces as `%20` in the final request to the gateway.** Encoding the browser-to-backend request isn't enough if the backend rebuilds the URL. Form-style query builders (JS `URLSearchParams`, Python `urlencode`, …) write spaces as `+`, which is rejected - percent-encode the keyword on its own (a component-level encoder such as JS `encodeURIComponent` or Python `quote`) and append it to the URL, instead of passing it through the query builder. Also transliterate (`Málaga` → `Malaga`, `Tromsø` → `Tromso`) and replace punctuation other than `-` `'` `.` with a space.
- **`countryCode` narrows a search, it can't replace `keyword`.** `keyword` still filters: `keyword=A&countryCode=ES` misses BCN.
- **Country names don't match.** `SPAIN` returns Port of Spain, not Spanish airports - map the country name to its ISO code and pass it as `countryCode` alongside a keyword.
- **Group by `address.cityCode`, not result order.** Airports can precede their city; `cityCode` needs `view=FULL`.
- **Key records by `id`, not by IATA code.** One IATA code can map to two records - `AMAD` (Barajas airport) and `CMAD` (Madrid city) both carry `iataCode: MAD`. Collapsing them by IATA code silently merges an airport and its city.
- **Pass the picked record downstream, not its label.** Whatever consumes the selection should get the stored `iataCode` and `subType`. Searching the label text again on submit can resolve to a different location than the one the user picked - re-searching `BIRMINGHAM` for a user who picked Birmingham, Alabama returns Birmingham UK's airport as the top result.
- **Coverage is limited to airport-served cities.** A `CITY` search returns almost only cities with a commercial airport. Destinations without airports (ski resorts, small coastal towns, rural areas - e.g. Antibes) return no results, so an empty response in a hotel or activity context may just mean the city has no airport.

## Errors

| HTTP | Code | Title | Typical cause |
| --- | --- | --- | --- |
| 400 | 477 | INVALID FORMAT | Malformed `keyword` |
| 400 | 572 | INVALID OPTION | Unknown query parameter, or any query parameter on by-ID lookup |
| 400 | 2781 | INVALID LENGTH | `keyword` empty or over 30 characters |
| 400 | 4926 | INVALID DATA RECEIVED | A `sort` other than `analytics.travelers.score` |
| 400 | 32171 | MANDATORY DATA MISSING | A required parameter is missing |
| 404 | 1797 | NOT FOUND | Unknown or lower-case `locationId` on by-ID lookup |
| 429 | - | TOO MANY REQUESTS | Rate limit for the API key exceeded |
| 500 | 141 | SYSTEM ERROR HAS OCCURRED | |

All errors use the envelope `{ "errors": [{ "status", "code", "title", "detail", "source": { "parameter" } }] }`.

## Troubleshooting

### Bracket parameters silently dropped
**Cause:** Some HTTP clients don't URL-encode `[` and `]` in query strings.
**Solution:** URL-encode brackets: `page[limit]` → `page%5Blimit%5D`, `page[offset]` → `page%5Boffset%5D`.

## Related skills

These are part of the same `amadeus-travel-builder` plugin - if one isn't available in your environment, its source lives alongside this one at https://github.com/AmadeusITGroup/amadeus-travel-toolkit/tree/main/plugins/amadeus-travel-builder/skills.

- [city-search](../city-search/SKILL.md) - city-only search that returns cities **regardless of airport coverage** (including airport-less towns like Antibes that this skill omits), and can expand a city into the airports serving it. Reach for it for a two-step city → airport picker, or when the target may have no airport.
- [nearest-airports](../nearest-airports/SKILL.md) - when you have a lat/lon rather than typed text and need airports.
- [flight-search](../flight-search/SKILL.md) - consumes the IATA codes this picker resolves as its `origins` / `destinations`; append `+` to an `AIRPORT` pick so the search stays on that airport - a bare airport code can also pull in nearby ones.
