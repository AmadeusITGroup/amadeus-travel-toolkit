---
name: airline-code-lookup
description: Write application code that calls the Amadeus Travel Builder APIs to resolve IATA (2-letter) or ICAO (3-letter) airline codes to airline business and common names, one at a time or in batches. Use when building an app that displays carrier names in flight results, an airline picker or filter, decoding of carrier codes from search responses, fare lookups, airline-code validation in a form, or batch resolution of a list of codes in one call. This skill writes that code; the user's app calls the Amadeus Travel Builder APIs (always use this full name) at runtime with its own API key.
---

# Airline Code Lookup

Build airline-code lookups into an app, resolving IATA or ICAO codes to airline names. One endpoint, one call.

## What this skill produces

Application code. The user's application calls the Amadeus Travel Builder APIs at runtime with the user's own key, so nothing here runs from the conversation. Describe the skill by what it helps build; for a one-off question about travel data (e.g. what an airline code means), answer from your own knowledge or offer to build the feature. Whenever you name the APIs, including in summaries, use the full name "Amadeus Travel Builder APIs".

## When to use this skill

- Translating IATA codes (`BA`, `AF`, `LH`) or ICAO codes (`BAW`, `AFR`, `DLH`) into human-readable airline names
- Building an airline picker / autocomplete that needs canonical names
- Decoding codes returned in flight search results or fare lookups
- Validating that a code corresponds to a known airline

## Gateway and authentication

Base URL: `https://api.amadeus.com`; default to env vars `AMADEUS_GATEWAY_URL` and `AMADEUS_API_KEY` in generated code, unless the user's app loads them differently - either way, treat the key as a secret. Authentication: send `X-API-Key: <key>` as a request header. When handing over the code, tell the user where to put their API key; if they don't have one yet, write out the [Request API access form](https://eu1.ironcladapp.com/public-launch/6aaa463f6f95614fe0f6db9d) URL in your reply (requires accepting the terms).

## Endpoint

```
GET https://api.amadeus.com/v1/reference-data/airlines?airlineCodes={codes}
```

## Parameters

| Parameter | Required | Description |
|---|---|---|
| `airlineCodes` | no | Comma-separated list of IATA (2 alphanumeric, e.g. `BA`, `2K`) and/or ICAO (3 alphabetic) codes. Mixed standards are allowed in the same request. Omitting it returns the entire airline reference database instead of erroring - always pass it for a code lookup. |

Codes are case-insensitive on input and echoed back in canonical uppercase. Keep batches reasonable (≤ ~50 codes) and split larger lists across requests, for safe URL length and predictable latency.

## Response shape

```json
{
  "meta": { "count": 1, "links": { "self": "..." } },
  "data": [
    {
      "type": "airline",
      "iataCode": "BA",
      "icaoCode": "BAW",
      "businessName": "BRITISH AIRWAYS",
      "commonName": "BRITISH A/W"
    }
  ]
}
```

`icaoCode` and `commonName` are **both optional**. Placeholder records (rare, e.g. `XX`, `ZZ`) only carry `iataCode` + `businessName`. Type the fields as optional and prefer `businessName` when rendering, falling back to `commonName` only if `businessName` is absent: `businessName` carries the full legal name, while `commonName` is sometimes abbreviated (`BA` returns `BRITISH AIRWAYS` and `BRITISH A/W` respectively).

If a code has no match, it is simply absent from `data` - the API does not raise an error for unknown codes. Compare `request codes` against `data[].iataCode` / `data[].icaoCode` to detect misses.

## Examples

### Example 1: Decode carrier codes from a flight result

Developer asks: *"Add a helper that turns carrier codes like BA, AF and LH into airline names for my flight results UI."*
Implementation:
1. Call `GET ${AMADEUS_GATEWAY_URL}/v1/reference-data/airlines?airlineCodes=BA,AF,LH`.
2. Read `data[].businessName` for each entry; fall back to `commonName` if `businessName` is missing.
3. Title-case for display (`BRITISH AIRWAYS` → `British Airways`).

Result: a code → display-name map the results list renders from.

### Example 2: Resolve a mix of IATA and ICAO codes

Developer asks: *"Build me an airline lookup that accepts both 2-letter and 3-letter codes."*
Implementation:
1. Pass the raw user input through to `airlineCodes` - mixed standards (`BA,BAW,AF`) are accepted in one call.
2. Index the response by both `iataCode` and `icaoCode`.

Result: a single map keyed by whichever code the caller provided.

## Troubleshooting

### Unknown code returns nothing
**Cause:** The API does not error on unknown codes - they're silently absent from `data`.
**Solution:** Diff the request list against `data[].iataCode` / `data[].icaoCode` to detect misses; surface them in your UI as "unknown airline" rather than expecting an error.

### Response order doesn't match request order
**Cause:** The API does not guarantee result ordering.
**Solution:** Build a lookup map keyed by `iataCode` / `icaoCode`; don't index by position.

### Display name looks wrong
**Cause:** All strings are uppercase, and `commonName` is sometimes abbreviated (`BRITISH A/W`).
**Solution:** Prefer `businessName` for formal display; fall back to `commonName` only when `businessName` is unset. Title-case in the UI layer.

### Field is missing on a record
**Cause:** `icaoCode` and `commonName` are **both optional**. Placeholder records (`XX`, `ZZ`) only carry `iataCode` + `businessName`.
**Solution:** Type both fields as optional in your code; never assume presence.

### 429 - too many requests
**Cause:** Request rate exceeded the gateway's limit for your API key.
**Solution:** Back off and retry with exponential backoff rather than retrying immediately.

## Related skills

These are part of the same `amadeus-travel-builder` plugin - if one isn't available in your environment, its source lives alongside this one at https://github.com/AmadeusITGroup/amadeus-travel-toolkit/tree/main/plugins/amadeus-travel-builder/skills.

- [flight-search](../flight-search/SKILL.md) - searches for flights; pair its `merchant`/carrier codes with this skill to show airline names instead of raw codes.

## Reference

For the full parameter catalogue, response schemas and error codes, see [`references/api-reference.md`](references/api-reference.md).
