---
name: airport-city-search
description: Write application code for a unified airport and city typeahead on the Amadeus Travel Builder APIs - keyword search across airports and cities by name or IATA code, ranked by traveller traffic, plus by-ID lookup. Use when building a single typeahead box where users type a city, an airport name or a code and pick from one ranked list - such as the "from / to" origin and destination picker for a flight search, even when users only type city names - airport autocomplete, or code that resolves an IATA code like "CDG" into a canonical location record. The API returns flat, sub-type-prefixed records covering airports and the cities they serve. This skill writes that code; the user's app calls the Amadeus Travel Builder APIs (always use this full name) at runtime with its own API key.
---

# Airport & City Search

Build keyword search and by-ID lookup over a unified set of airports and cities - the typeahead behind most "where are you flying from / to?" pickers.

## What this skill produces

Application code. The user's application calls the Amadeus Travel Builder APIs at runtime with the user's own key, so nothing here runs from the conversation. Describe the skill by what it helps build; for a one-off question about travel data (e.g. which airport a typed city or airport name refers to), answer from your own knowledge or offer to build the feature. Whenever you name the APIs, including in summaries, use the full name "Amadeus Travel Builder APIs".

## When to use this skill

- Building a flight-booking origin / destination picker
- Airport autocomplete (codes, names, terminal hints)
- Unified airport-and-city search where users may type either ("Paris" or "CDG")
- Resolving a previously stored `locationId` back to its full record

The API only returns locations that have at least one commercial airport. Smaller cities, resort towns, and rural destinations without airports will return no results.

## Gateway and authentication

Base URL: `https://api.amadeus.com`; default to env vars `AMADEUS_GATEWAY_URL` and `AMADEUS_API_KEY` in generated code, unless the user's app loads them differently - either way, treat the key as a secret. Authentication: send `X-API-Key: <key>` as a request header. When handing over the code, tell the user where to put their API key; if they don't have one yet, write out the [Request API access form](https://eu1.ironcladapp.com/public-launch/6aaa463f6f95614fe0f6db9d) URL in your reply (requires accepting the terms).

## Endpoints

### Keyword search

```
GET https://api.amadeus.com/v1/reference-data/locations?subType={AIRPORT|CITY|AIRPORT,CITY}&keyword={keyword}
```

### By-ID lookup

```
GET https://api.amadeus.com/v1/reference-data/locations/{locationId}
```

## Keyword search parameters

| Parameter | Required | Default | Notes |
|---|---|---|---|
| `subType` | yes | - | `AIRPORT`, `CITY`, or `AIRPORT,CITY`. Drives what comes back. |
| `keyword` | yes | - | Matches the start of any word in the location name **or** the start of an IATA code. At least one character. |
| `countryCode` | no | - | ISO 3166-1 alpha-2 (e.g. `US`, `FR`) to narrow a keyword search by country. |
| `view` | no | `FULL` | `LIGHT` for a label-only autocomplete (smaller payload, no `geoCode` or `address.cityCode`). `FULL` when the pick feeds a map, groups airports under their city, or is passed to another call. |
| `page[limit]` | no | `10` | Max items per page. URL-encode brackets in clients that require it (`page%5Blimit%5D`). |
| `page[offset]` | no | `0` | Page offset. |
| `sort` | no | `analytics.travelers.score` | The only legal value - the API exposes a `sort` param but doesn't actually support alternatives; send this exact string or omit it. |

## By-ID lookup parameters

| Parameter | Required | Notes |
|---|---|---|
| `locationId` *(path)* | yes | Sub-type-prefixed identifier returned by the keyword search - `A...` for airports (`ALHR` = Heathrow), `C...` for cities (`CPAR` = Paris). |

## Response shape - keyword search

```json
{
  "meta": {
    "count": 127,
    "links": {
      "self":  "https://api.amadeus.com/v1/reference-data/locations?subType=AIRPORT,CITY&keyword=lon&page[limit]=2",
      "next":  "...&page[offset]=2&page[limit]=2"
    }
  },
  "data": [
    {
      "type": "location",
      "subType": "AIRPORT",
      "name": "HEATHROW",
      "detailedName": "LONDON/GB:HEATHROW",
      "id": "ALHR",
      "self": { "href": "...", "methods": ["GET"] },
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
      "analytics": { "travelers": { "score": 45 } }
    }
  ]
}
```

`view=LIGHT` returns a trimmed payload - at minimum `iataCode`, `name`, `detailedName`, and a partial `address` (`{ cityName, countryName }`). In practice it also returns `id`, `self`, `type`, and `subType`. Type against that full shape, not the shorter list. `view=FULL` adds `analytics`, `geoCode` and `timeZoneOffset`.

## Response shape - by-ID lookup

A single object under `data`, identical to the keyword-search records above but unwrapped from the array:

```json
{
  "meta": { "links": { "self": "..." } },
  "data": {
    "type": "location",
    "subType": "AIRPORT",
    "id": "ALHR",
    ...
  }
}
```

## Examples

### Example 1: From / to airport typeahead

Developer asks: *"Add an airport picker to my flight form."*
Implementation:
1. Debounce input (~250ms), transliterate it to ASCII and percent-encode it (spaces as `%20`), then call `GET ${AMADEUS_GATEWAY_URL}/v1/reference-data/locations?subType=AIRPORT,CITY&keyword={input}&view=LIGHT&page%5Blimit%5D=8`.
2. Render `name` (or `detailedName`) plus `iataCode`, with a `subType` badge (`AIRPORT` vs `CITY`).
3. Store the selected record - its `id` (sub-type-prefixed, e.g. `ALHR`), `iataCode` and `subType` - in your form state, not just the label shown in the input.

Result: a typeahead that resolves user text into IATA codes and canonical location records.

### Example 2: Resolve a stored selection back to its full record

Developer asks: *"Add a recent-airport view that reloads the full record for the airport a user picked last time."*
Implementation:
1. Read the stored `id` (e.g. `ALHR`).
2. Call `GET ${AMADEUS_GATEWAY_URL}/v1/reference-data/locations/ALHR`.
3. Render the unwrapped `data` object (no array - single record).

Result: code that restores a stored selection to its full record.

### Example 3: Country-scoped city picker

Developer asks: *"Let users search for German cities only."*
Implementation:
1. Call with `subType=CITY&countryCode=DE&keyword={input}`.
2. Render results.

Result: typeahead constrained to German cities.

## Things to get right

- **Always send `subType`.** It's required - omitting it returns a 400 - so pass at least one of `AIRPORT`, `CITY`, or both as CSV on every keyword search.
- **Send `keyword` as ASCII, with spaces as `%20`.** Form-style query encoders write spaces as `+`, which is rejected - percent-encode it, transliterate (`Málaga` → `Malaga`, `Tromsø` → `Tromso`) and replace punctuation other than `-` `'` `.` with a space.
- **`countryCode` narrows a search, it can't replace `keyword`.** `keyword` needs at least one character and still filters: `keyword=A&countryCode=ES` misses BCN.
- **Country names don't match.** `SPAIN` returns Port of Spain, not Spanish airports - map the country name to its ISO code and pass it as `countryCode` alongside a keyword.
- **Group by `address.cityCode`, not result order.** Airports can precede their city; `cityCode` needs `view=FULL`.
- **Key records by `id`, not by IATA code.** The `id` is sub-type-prefixed, so one IATA code can map to two records - `AMAD` (Barajas airport) and `CMAD` (Madrid city) both carry `iataCode: MAD`. Collapsing them by IATA code alone silently merges an airport and its city.
- **Pass the picked record downstream, not its label.** Whatever consumes the selection should get the stored `iataCode` and `subType`. Searching the label text again on submit can resolve to a different location than the one the user picked - city names especially: re-searching `BIRMINGHAM` for a user who picked Birmingham, Alabama returns Birmingham UK's airport as the top result.
- **Coverage is limited to airport-served cities.** A `CITY` subType search only returns cities with at least one commercial airport. Destinations without airports (ski resorts, small coastal towns, rural areas - e.g. Antibes) return no results - don't treat an empty response as "city not found" in a general hotel or activity search context; the city may simply have no airport. When you need those cities, use the [city-search](../city-search/SKILL.md) skill, which returns cities regardless of airport coverage.

## Troubleshooting

### 400 - missing `subType`
**Cause:** `subType` is required.
**Solution:** Always pass `subType=AIRPORT`, `subType=CITY`, or `subType=AIRPORT,CITY`.

### 400 - `477 INVALID FORMAT` on `keyword`
**Cause:** The keyword contains a `+`-encoded space, a non-ASCII letter, or punctuation such as `,` `&` `?`.
**Solution:** Percent-encode spaces as `%20`, not form-style `+`, transliterate to ASCII and replace punctuation other than `-` `'` `.` with a space.

### Bracket parameters silently dropped
**Cause:** Some HTTP clients don't URL-encode `[` and `]` in query strings.
**Solution:** URL-encode brackets: `page[limit]` → `page%5Blimit%5D`, `page[offset]` → `page%5Boffset%5D`.

### 400 - custom `sort` parameter rejected
**Cause:** `analytics.travelers.score` is the only accepted value; anything else returns a 400 (`INVALID DATA RECEIVED`), not a silently-ignored sort.
**Solution:** Don't pass a `sort` parameter; rely on the default.

### 404 on `locationId` lookup
**Cause:** Unknown ID returns `1797 NOT FOUND`.
**Solution:** Handle this code distinctly from 400 (validation) - render a "not found" UI rather than a "bad request" error.

### `address.countryCode` is undefined
**Cause:** You're on `view=LIGHT`, which trims `address` down to just `cityName` and `countryName`.
**Solution:** Use `view=FULL` (the default) - its `address` carries `countryCode`, `cityCode` and `regionCode` too.

### 429 - too many requests
**Cause:** Request rate exceeded the gateway's limit for your API key.
**Solution:** Back off and retry with exponential backoff rather than retrying immediately.

## Related skills

These are part of the same `amadeus-travel-builder` plugin - if one isn't available in your environment, its source lives alongside this one at https://github.com/AmadeusITGroup/amadeus-travel-toolkit/tree/main/plugins/amadeus-travel-builder/skills.

- [city-search](../city-search/SKILL.md) - city-only search that returns cities **regardless of airport coverage** (including airport-less towns like Antibes that this skill omits), and can expand a city into the airports serving it. Reach for it for a two-step city → airport picker, or when the target may have no airport.
- [nearest-airports](../nearest-airports/SKILL.md) - when you have a lat/lon rather than typed text and need airports.
- [flight-search](../flight-search/SKILL.md) - consumes the IATA codes this picker resolves as its `origins` / `destinations`; append `+` to an `AIRPORT` pick so the search stays on that airport - a bare airport code can also pull in nearby ones.

## Reference

For the full parameter catalogue, response schemas and error codes, see [`references/api-reference.md`](references/api-reference.md).
