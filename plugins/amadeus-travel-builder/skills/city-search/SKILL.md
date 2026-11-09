---
name: city-search
description: Write application code that calls the Amadeus Travel Builder APIs to match a typed city keyword (optionally scoped to a country) to cities with their IATA city code, coordinates, and, on request, the airports that serve each city - including cities that have no airport of their own (e.g. Antibes, resort/rural towns). Use when building a city autocomplete or country-scoped city lookup for destination guides, hotels or activities, code that resolves a typed city name like "Paris" or "Lyon" into its coordinates for a location-based search, or a two-step city → airport picker where the user chooses a city and then one of the airports serving it. When the app needs cities regardless of airport coverage, this is the skill. Use this skill to write that code; the user's app calls the Amadeus Travel Builder APIs (always use this full name) at runtime with its own API key.
---

# City Search

Build city search by keyword into an app. A user types part of a city name - optionally scoped to a country - and your app gets back matching cities with their IATA city code and coordinates. Ask for `include=AIRPORTS` and each city also carries the airports that serve it.

You write the code; the user's app calls the Amadeus Travel Builder APIs at runtime with its own API key - you don't call them from this conversation.

## Gateway and authentication

Base URL: `https://api.amadeus.com`; default to env vars `AMADEUS_GATEWAY_URL` and `AMADEUS_API_KEY` in generated code, unless the user's app loads them differently - either way, treat the key as a secret, keeping it out of client code and publicly served files. Authentication: send `X-API-Key: <key>` as a request header. When handing over the code, tell the user where to put their API key; if they don't have one yet, write out the [Request API access form](https://eu1.ironcladapp.com/public-launch/6aaa463f6f95614fe0f6db9d) URL in your reply (requires accepting the terms).

## Endpoint

```
GET https://api.amadeus.com/v1/reference-data/locations/cities?keyword={keyword}&countryCode={cc}&include=AIRPORTS
```

## Parameters

| Parameter | Required | Type | Constraints | Notes |
| --- | --- | --- | --- | --- |
| `keyword` | yes | string | 3-50 chars | Case-insensitive **substring** of the city name (e.g. `PARIS`, `LYO`). Letters and inner spaces only - no digits, hyphens, apostrophes, dots or `*`; a leading or trailing space matches nothing. |
| `countryCode` | no | string | exactly 2 chars | ISO 3166 Alpha-2 (e.g. `US`, `FR`); scopes results to one country. |
| `max` | no | integer | `1`-`1000` | Cap on the number of cities returned. Default and maximum `1000`; out-of-range values are ignored with warning `111`. |
| `include` | no | enum | `AIRPORTS` | Case-sensitive. Adds `relationships` on each city and a top-level `included.airports` map. |

## Response shape

```json
{
  "data": [
    {
      "type": "location",
      "subType": "city",
      "name": "Paris",
      "iataCode": "PAR",
      "address": { "countryCode": "FR", "stateCode": "FR-75" },
      "geoCode": { "latitude": 48.85341, "longitude": 2.3488 },
      "relationships": [
        { "id": "BVA", "type": "Airport", "href": "#/included/airports/BVA" },
        { "id": "CDG", "type": "Airport", "href": "#/included/airports/CDG" },
        { "id": "ORY", "type": "Airport", "href": "#/included/airports/ORY" },
        { "id": "XCR", "type": "Airport", "href": "#/included/airports/XCR" }
      ]
    }
  ],
  "included": {
    "airports": {
      "BVA": { "subType": "Airport", "name": "Beauvais-Tille", "iataCode": "BVA", "address": { "countryCode": "FR", "stateCode": "FR-75" }, "geoCode": { "latitude": 49.45444, "longitude": 2.11278 } },
      "CDG": { "subType": "Airport", "name": "Charles de Gaulle", "iataCode": "CDG", "address": { "countryCode": "FR", "stateCode": "FR-75" }, "geoCode": { "latitude": 49.01278, "longitude": 2.55 } },
      "ORY": { "subType": "Airport", "name": "Orly", "iataCode": "ORY", "address": { "countryCode": "FR", "stateCode": "FR-75" }, "geoCode": { "latitude": 48.72528, "longitude": 2.35944 } },
      "XCR": { "subType": "Airport", "name": "Chalons-Vatry", "iataCode": "XCR", "address": { "countryCode": "FR", "stateCode": "FR-75" }, "geoCode": { "latitude": 48.77611, "longitude": 4.18444 } }
    }
  },
  "meta": { "count": 8, "links": { "self": "..." } }
}
```
The response to `keyword=PARIS&countryCode=FR&include=AIRPORTS`, trimmed to the first of its 8 cities and that city's airports.

The content type is `application/vnd.amadeus+json` whatever `Accept` you send; requesting `application/json` is fine.

### City record

| Field | Type | Notes |
| --- | --- | --- |
| `type` | string | `"location"`. |
| `subType` | string | `"city"`. |
| `name` | string | Mixed case, e.g. `Paris`. |
| `iataCode` | string, optional | IATA **city** code, e.g. `PAR`. |
| `address.countryCode` | string | ISO 3166-1 alpha-2, uppercase. |
| `address.stateCode` | string, optional | `{countryCode}-{subdivision}`, e.g. `FR-75`, `US-IL`; `{countryCode}-ZZZ` (e.g. `IT-ZZZ`) in countries stored without subdivisions. |
| `geoCode` | `{ latitude?, longitude? }` | Floats; can be `{}`. |
| `relationships` | `{ id, type: "Airport", href }[]` | Only with `include=AIRPORTS`. `id` is the airport's key in `included.airports`; `href` is a JSON-pointer-style reference such as `#/included/airports/CDG`. |

### Other top-level keys

| Key | Type | Notes |
| --- | --- | --- |
| `included.airports` | map of IATA code → `{ subType: "Airport", name, iataCode, address, geoCode }` | Only with `include=AIRPORTS`. An airport's `address.stateCode` is that of the city it serves, not its own location - Chalons-Vatry (`XCR`) carries Paris's `FR-75`. |
| `meta.count` | integer | Number of cities in `data` (at most 1000), not the total number of matches. |
| `meta.links.self` | string | Link to the current query. |
| `warnings` | `{ status, code, title, detail?, source? }[]` | Optional, with HTTP 200; not a failure. `8200` means more matches exist than were returned - narrow with `countryCode`. |

## Examples

### Example 1: Country-scoped city autocomplete

Developer asks: *"Let users search for French cities as they type."*
Implementation:
1. For a worked example, the user types `NIC`: transliterate the input to ASCII and call `GET ${AMADEUS_GATEWAY_URL}/v1/reference-data/locations/cities?keyword=NIC&countryCode=FR&max=10`. Nice (`NCE`) comes back first, alongside towns such as Pornic and Pornichet that have no code.
2. Render `name` plus `iataCode` (when present) for each result.
3. Store the selection's `iataCode` (the IATA city code) in form state.

Result: a typeahead constrained to French cities that resolves typed text into IATA city codes.

### Example 2: Expand a city into its airports

Developer asks: *"When someone picks a city, let them choose which of its airports to fly from."*
Implementation:
1. For a worked example, Paris: call `GET ${AMADEUS_GATEWAY_URL}/v1/reference-data/locations/cities?keyword=PARIS&countryCode=FR&include=AIRPORTS`.
2. For the chosen city, walk `relationships[]`; for each entry, look up `included.airports[relationship.id]` to get the airport's `name` and `iataCode`.
3. Render the airports - for Paris, Beauvais-Tille (`BVA`), Charles de Gaulle (`CDG`), Orly (`ORY`) and Chalons-Vatry (`XCR`) - as sub-options under the city.

Result: a city → airports drill-down for a flight-origin selector.

### Example 3: Resolve a city name to its IATA code

Developer asks: *"Write code that resolves a typed city name to its IATA city code."*
Implementation:
1. For a worked example, "Lyon": call with `keyword=LYO&countryCode=FR`. Lyon (`LYS`) comes back alongside towns such as Sainte-Foy-les-Lyon that have no code.
2. Pick the record whose `name` best matches the input rather than taking `data[0]`, and skip records without an `iataCode`.
3. Read `iataCode` from the chosen record.

Result: a name → city-code resolver that skips records without a code.

## Things to get right

- **Airports come back cross-referenced, not inline.** Resolve each `relationships[].id` through `included.airports` - treating `relationships` as the airport records loses `name`/`iataCode`.
- **`included.airports` is one shared pool for the whole response, not per-city.** When `data` holds several cities (which the substring match makes common), it's the union of every city's airports. Select a city's airports through *its own* `relationships[].id` - rendering the whole map under one city wrongly attributes other cities' airports to it.
- **`keyword` is a substring match.** It matches anywhere in the name, so `PARIS` returns not just Paris but Le Touquet-Paris-Plage, Villeparisis, Damparis, Parisot and others. Rank/filter by name yourself (prefer exact and word-start matches) rather than assuming `data[0]` is the city the user meant, and scope with `countryCode` to cut noise. Validate the final `keyword` after transliteration and word selection; send it only when it meets the 3-50-character limit, since normalization can leave a shorter fragment even when the original input is long enough.
- **`iataCode` is often absent** - the key is missing, not `null`. If you're resolving a name to a code, skip records without one. In a picker that feeds a location-based search (activities, points of interest), keep them - key the downstream request on `geoCode` or `name` when there's no `iataCode`, since filtering them out drops exactly the airport-less towns this API exists to return. A city can also have a non-airport transport code - Antibes returns `XAT`, a rail/coach code - so don't assume `iataCode` implies an airport.
- **`geoCode` can be an empty object `{}`** - check `geoCode?.latitude` before using coordinates.
- **The same town can come back twice, with complementary halves.** Antibes returns one record with `iataCode: XAT` but empty `geoCode`, and another with coordinates but no `iataCode`. Merge records with the same `name` + `address.countryCode` into one (code from one, coordinates from the other) rather than listing duplicates or dropping one.
- **No `id` on records.** Key them on `name` + `address.countryCode` (+ `iataCode` when present).
- **Send `keyword` as ASCII.** An accented keyword (`Málaga`) returns a `1797 DATA NOT FOUND` warning at HTTP 200, not an error - transliterate (`Malaga`) before calling and percent-encode spaces as `%20`. Stored names keep their hyphens, which the keyword can't contain, so `SAINT MALO` finds nothing - search one word of the name (`MALO`) and match the full name client-side.
- **Cities are stored under their current names.** `BANGALORE` finds nothing; `BENGALURU` returns Bengaluru (`BLR`).
- **No match has no `data` key.** The response is HTTP 200 with `meta.count: 0` and the `1797` warning - read `data` with a default of `[]`.
- **Towns without their own airport usually link to the airports serving them.** With `include=AIRPORTS`, the Antibes record with coordinates links to `NCE`; only its coded twin (`XAT`) has no `relationships`. A city with no links is a normal case, not an error.

## Errors

| HTTP | Code | Title | Typical cause |
| --- | --- | --- | --- |
| 400 | 32171 | MANDATORY DATA MISSING | `keyword` omitted |
| 400 | 572 | INVALID OPTION | Unknown query parameter |
| 400 | 2781 | INVALID LENGTH | `keyword` outside 3-50 chars, or `countryCode` not 2 chars |
| 400 | 477 | INVALID FORMAT | A character other than letters and spaces in `keyword`, or a non-numeric `max` |
| 400 | 4926 | INVALID DATA RECEIVED | `include` other than `AIRPORTS` |
| 429 | - | TOO MANY REQUESTS | Rate limit for the API key exceeded |
| 500 | 141 | SYSTEM ERROR HAS OCCURRED | |

All errors use the envelope `{ "errors": [{ "status", "code", "title", "detail" }] }`.

## Troubleshooting

### No results for a valid-looking city
**Cause:** The keyword didn't match any substring of a known city name, or the `countryCode` scope excluded it.
**Solution:** Widen by dropping `countryCode`, or shorten the keyword to a cleaner fragment.

## Related skills

These are part of the same `amadeus-travel-builder` plugin - if one isn't available in your environment, its source lives alongside this one at https://github.com/AmadeusITGroup/amadeus-travel-toolkit/tree/main/plugins/amadeus-travel-builder/skills.

- [airport-city-search](../airport-city-search/SKILL.md) - unified airport **and** city typeahead returning flat, sub-type-prefixed location records; use it for a single ranked typeahead where users may type a city, an airport name or a code. It returns almost only cities that have a commercial airport, so stay with city-search when you need cities regardless of airport coverage.
- [nearest-airports](../nearest-airports/SKILL.md) - when you have a lat/lon rather than a typed city name and need airports.
- [flight-search](../flight-search/SKILL.md) - the IATA city codes this skill resolves feed directly into its `origins` / `destinations`.
