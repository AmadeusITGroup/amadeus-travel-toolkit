---
name: city-search
description: Write application code that calls the Amadeus Travel Builder APIs to match a typed city keyword (optionally scoped to a country) to cities with their IATA city code, coordinates, and, on request, the airports that serve each city - including cities that have no airport of their own (e.g. Antibes, resort/rural towns). Use when building a city autocomplete or country-scoped city lookup for destination guides, hotels or activities, code that resolves a typed city name like "Paris" or "Bangalore" into its coordinates for a location-based search, or a two-step city → airport picker where the user chooses a city and then one of the airports serving it. When the app needs cities regardless of airport coverage, this is the skill. This skill writes that code; the user's app calls the Amadeus Travel Builder APIs (always use this full name) at runtime with its own API key.
---

# City Search

Build city search by keyword into an app. A user types part of a city name - optionally scoped to a country - and your app gets back matching cities with their IATA city code and coordinates. Ask for `include=AIRPORTS` and each city also carries the airports that serve it.

## What this skill produces

Application code. The user's application calls the Amadeus Travel Builder APIs at runtime with the user's own key, so nothing here runs from the conversation. Describe the skill by what it helps build; for a one-off question about travel data (e.g. what city a name or keyword resolves to), answer from your own knowledge or offer to build the feature. Whenever you name the APIs, including in summaries, use the full name "Amadeus Travel Builder APIs".

## When to use this skill

- City autocomplete / picker where users type a city name
- Resolving a typed city ("Paris", "Bangalore") into its IATA city code (`PAR`, `BLR`)
- Country-scoped city lookup (only cities in `FR`, only cities in `US`)
- Mapping a city to the airports within it - e.g. to expand a chosen city into selectable airports for a flight-origin field
- City lookup that must work even for cities without their own airport - e.g. Antibes or other resort/rural towns, which [airport-city-search](../airport-city-search/SKILL.md) returns nothing for (it only surfaces airport-served cities)

## Gateway and authentication

Base URL: `https://api.amadeus.com`; default to env vars `AMADEUS_GATEWAY_URL` and `AMADEUS_API_KEY` in generated code, unless the user's app loads them differently - either way, treat the key as a secret. Authentication: send `X-API-Key: <key>` as a request header. When handing over the code, tell the user where to put their API key; if they don't have one yet, write out the [Request API access form](https://eu1.ironcladapp.com/public-launch/6aaa463f6f95614fe0f6db9d) URL in your reply (requires accepting the terms).

## Endpoint

```
GET https://api.amadeus.com/v1/reference-data/locations/cities?keyword={keyword}&countryCode={cc}&include=AIRPORTS
```

## Parameters

| Parameter | Required | Default | Notes |
|---|---|---|---|
| `keyword` | yes | - | Case-insensitive substring of the city name (e.g. `PARIS`, `BANG`). 2–10 characters. Matches anywhere in the name, so `PARIS` also returns Villeparisis, Damparis, etc. |
| `countryCode` | no | - | ISO 3166 Alpha-2 (e.g. `US`, `FR`) to scope results to one country. |
| `max` | no | - | Cap on the number of cities returned. |
| `include` | no | - | Set to `AIRPORTS` to also return the airports serving each city. Cross-referenced via `relationships` (see below). |

## Response shape

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

Each city record carries `name`, `iataCode` (the IATA **city** code), `address.countryCode`, and `geoCode`. When you pass `include=AIRPORTS`, the airports don't sit inline on the city - they live in the top-level `included.airports` map, keyed by IATA code, and each city links them through its `relationships[]` array (`id` = the airport's key in `included.airports`). Without `include=AIRPORTS`, there is no `relationships` array and no `included` block.

## Examples

### Example 1: Country-scoped city autocomplete

Developer asks: *"Let users search for French cities as they type."*
Implementation:
1. Debounce input (~250ms), then call `GET ${AMADEUS_GATEWAY_URL}/v1/reference-data/locations/cities?keyword={input}&countryCode=FR&max=10`.
2. Render `name` plus `iataCode` for each result.
3. Store the selection's `iataCode` (the IATA city code) in form state.

Result: a typeahead constrained to French cities that resolves typed text into IATA city codes.

### Example 2: Expand a city into its airports

Developer asks: *"When someone picks a city, let them choose which of its airports to fly from."*
Implementation:
1. For a worked example, Paris: call `GET ${AMADEUS_GATEWAY_URL}/v1/reference-data/locations/cities?keyword=PARIS&countryCode=FR&include=AIRPORTS`.
2. For the chosen city, walk `relationships[]`; for each entry, look up `included.airports[relationship.id]` to get the airport's `name` and `iataCode`.
3. Render the airports (CDG, ORY, …) as sub-options under the city.

Result: a city → airports drill-down for a flight-origin selector.

### Example 3: Resolve a city name to its IATA code

Developer asks: *"Write code that resolves a typed city name to its IATA city code."*
Implementation:
1. For a worked example, "Bangalore": call with `keyword=BANG` (optionally `countryCode=IN` to disambiguate).
2. Because the match is a substring, `data` may include unrelated towns - pick the record whose `name` best matches the input (prefer an exact/word-start match) rather than blindly taking `data[0]`, and skip records where `iataCode` is `null`.
3. Read `iataCode` from the chosen record.

Result: code that resolves a typed city name to its IATA city code.

## Things to get right

- **Airports come back cross-referenced, not inline.** With `include=AIRPORTS`, each city's `relationships[].id` is a key into the top-level `included.airports` map - the airport objects are not nested on the city. Resolve them by looking up `included.airports[id]`; treating `relationships` as the airport records themselves loses `name`/`iataCode`.
- **`included.airports` is one shared pool for the whole response, not per-city.** When `data` holds several cities (which the substring match makes common), `included.airports` is the union of every city's airports. Always select a given city's airports through *its own* `relationships[].id` - rendering the whole map under one city wrongly attributes other cities' airports to it.
- **`keyword` is a substring match, min 2 chars.** It matches anywhere in the city name, case-insensitively - so `PARIS` returns not just Paris but Villeparisis, Damparis, Parisot, and other names that merely contain the letters. Rank/filter results by name yourself (e.g. prefer exact and word-start matches) rather than assuming `data[0]` is the city the user meant; scope with `countryCode` to cut noise.
- **`iataCode` is often `null`.** Only cities with an IATA city code carry one; many smaller towns the substring match pulls in have `iataCode: null`. If you're resolving a name to a code, skip records without one rather than rendering `null`. In a picker that feeds a location-based search (activities, points of interest), keep them - key the downstream request on `geoCode` or `name` when `iataCode` is `null`, since filtering them out drops exactly the airport-less towns this API exists to return. (A city can also have a non-airport transport code - e.g. Antibes returns `XAT`, a rail/coach code - so don't assume `iataCode` implies an airport.)
- **Airport-less cities return no airports even with `include=AIRPORTS`.** A city with no commercial airport (Antibes, most resort/rural towns) comes back with no `relationships` and contributes nothing to `included.airports`. That's expected, not an error - this API still returns the *city* (unlike airport-city-search, which omits it entirely); there simply are no airports to expand into.

## Troubleshooting

### 400 - MANDATORY DATA MISSING (code 32171)
**Cause:** `keyword` is required and was omitted.
**Solution:** Always send a non-empty `keyword` (2–10 chars).

### 400 - INVALID LENGTH (code 2781)
**Cause:** `keyword` shorter than 2 or longer than 10 characters, or `countryCode` not exactly 2 characters.
**Solution:** Enforce the length bounds client-side before calling.

### 400 - INVALID OPTION (code 572)
**Cause:** `include` set to an unsupported value.
**Solution:** The only supported value is `AIRPORTS`.

### No airports on the city record
**Cause:** `include=AIRPORTS` wasn't sent, so there's no `relationships`/`included` data.
**Solution:** Add `include=AIRPORTS`, then resolve airports via `included.airports`.

### Empty `data` for a valid-looking city
**Cause:** The keyword didn't match any substring of a known city name, or the `countryCode` scope excluded it.
**Solution:** Widen by dropping `countryCode`, or shorten the keyword to a cleaner fragment.

### Unexpected cities in the results
**Cause:** `keyword` is a substring match, so `PARIS` also returns Villeparisis, Damparis, Parisot, etc.
**Solution:** Filter/rank by `name` client-side (prefer exact and word-start matches) and scope with `countryCode`.

### 429 - too many requests
**Cause:** Request rate exceeded the gateway's limit for your API key.
**Solution:** Back off and retry with exponential backoff rather than retrying immediately.

## Related skills

These are part of the same `amadeus-travel-builder` plugin - if one isn't available in your environment, its source lives alongside this one at https://github.com/AmadeusITGroup/amadeus-travel-toolkit/tree/main/plugins/amadeus-travel-builder/skills.

- [airport-city-search](../airport-city-search/SKILL.md) - unified airport **and** city typeahead returning flat, sub-type-prefixed location records; use it for a single ranked typeahead where users may type a city, an airport name or a code. It only returns cities that have at least one commercial airport, so stay with city-search when you need cities regardless of airport coverage.
- [nearest-airports](../nearest-airports/SKILL.md) - when you have a lat/lon rather than a typed city name and need airports.
- [flight-search](../flight-search/SKILL.md) - the IATA city codes this skill resolves feed directly into its `origins` / `destinations`.

## Reference

For the full parameter catalogue, response schemas and error codes, see [`references/api-reference.md`](references/api-reference.md).
