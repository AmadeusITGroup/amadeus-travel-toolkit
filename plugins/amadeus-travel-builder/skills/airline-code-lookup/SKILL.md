---
name: airline-code-lookup
description: Write application code that calls the Amadeus Travel Builder APIs to resolve IATA (2-letter) or ICAO (3-letter) airline codes to airline business and common names, one at a time or in batches. Use when building an app that displays carrier names in flight results, an airline picker or filter, decoding of carrier codes from search responses, fare lookups, airline-code validation in a form, or batch resolution of a list of codes in one call. Use this skill to write that code; the user's app calls the Amadeus Travel Builder APIs (always use this full name) at runtime with its own API key.
---

# Airline Code Lookup

Build airline-code lookups into an app, resolving IATA or ICAO codes to airline names. One endpoint, one call.

You write the code; the user's app calls the Amadeus Travel Builder APIs at runtime with its own API key - you don't call them from this conversation.

## Gateway and authentication

Base URL: `https://api.amadeus.com`; default to env vars `AMADEUS_GATEWAY_URL` and `AMADEUS_API_KEY` in generated code, unless the user's app loads them differently - either way, treat the key as a secret, keeping it out of client code and publicly served files. Authentication: send `X-API-Key: <key>` as a request header. When handing over the code, tell the user where to put their API key; if they don't have one yet, write out the [Request API access form](https://eu1.ironcladapp.com/public-launch/6aaa463f6f95614fe0f6db9d) URL in your reply (requires accepting the terms).

## Endpoint

```
GET https://api.amadeus.com/v1/reference-data/airlines?airlineCodes=BA,AFR,LH
```

The codes are illustrations of runtime input.

## Parameters

| Parameter | Required | Notes |
| --- | --- | --- |
| `airlineCodes` | no - but always send it | Comma-separated, no spaces: IATA (2 alphanumeric, e.g. `BA`, `2K`) and/or ICAO (3 alphabetic, e.g. `BAW`) codes, mixed freely - each is identified by its length. Case-insensitive; echoed back in uppercase. Omitting it returns every airline that has an IATA code (about 1,200), not ICAO-only airlines such as `VJT`. |

Keep batches to about 50 codes and split larger lists across requests, for safe URL length and predictable latency.

## Response shape

```json
{
  "meta": { "count": 3, "links": { "self": "https://api.amadeus.com/v1/reference-data/airlines?airlineCodes=BA,AFR,LH" } },
  "data": [
    { "type": "airline", "iataCode": "AF", "icaoCode": "AFR", "businessName": "AIR FRANCE", "commonName": "AIR FRANCE" },
    { "type": "airline", "iataCode": "BA", "icaoCode": "BAW", "businessName": "BRITISH AIRWAYS", "commonName": "BRITISH A/W" },
    { "type": "airline", "iataCode": "LH", "icaoCode": "DLH", "businessName": "LUFTHANSA", "commonName": "LUFTHANSA" }
  ]
}
```
The response to the request above.

| Field | Type | Notes |
| --- | --- | --- |
| `type` | `"airline"` | Constant. |
| `iataCode` | string, optional | Absent on airlines with no IATA code. |
| `icaoCode` | string, optional | |
| `businessName` | string | Always present; uppercase full legal name. `UNDEFINED` on placeholder records (e.g. `XX`, `9W`) - treat those as an unknown airline. |
| `commonName` | string, optional | Uppercase; sometimes abbreviated (`BRITISH A/W`). |

## Examples

### Example 1: Decode carrier codes from a flight result

Developer asks: *"Add a helper that turns carrier codes into airline names for my flight results UI."*
Implementation:
1. Call `GET ${AMADEUS_GATEWAY_URL}/v1/reference-data/airlines?airlineCodes=BA,AF,LH`.
2. Read `data[].businessName` for each entry; when it is `UNDEFINED`, use `commonName` if present, else treat the code as unknown.

Result: a code → display-name map the results list renders from.

### Example 2: Resolve a mix of IATA and ICAO codes

Developer asks: *"Build me an airline lookup that accepts both 2-letter and 3-letter codes."*
Implementation:
1. Remove spaces from the user input and pass it to `airlineCodes` - mixed standards (`BA,BAW,AF`) are accepted in one call, but a code after a space is silently dropped (`BA, AF` returns only BA).
2. Index the response by both `iataCode` and `icaoCode`.

Result: a single map keyed by whichever code the caller provided.

## Errors

| HTTP | Code | Meaning |
| --- | --- | --- |
| 400 | 572 `INVALID OPTION` | Unknown query parameter. |
| 429 | - | Rate limit for the API key exceeded. |
| 500 | 141 `SYSTEM ERROR HAS OCCURRED` | Upstream failure. |

## Troubleshooting

### Unknown code returns nothing
**Cause:** Unknown and malformed codes don't error - they're silently absent from `data`.
**Solution:** Diff the requested codes against `data[].iataCode` / `data[].icaoCode` to find misses:

```ts
const found = new Set(response.data.flatMap(a => [a.iataCode, a.icaoCode].filter(Boolean)))
const missing = requested.filter(code => !found.has(code.toUpperCase()))
```

### Response order doesn't match request order
**Cause:** Result order isn't guaranteed.
**Solution:** Build a lookup map keyed by `iataCode` / `icaoCode`; don't index by position.

## Related skills

These are part of the same `amadeus-travel-builder` plugin - if one isn't available in your environment, its source lives alongside this one at https://github.com/AmadeusITGroup/amadeus-travel-toolkit/tree/main/plugins/amadeus-travel-builder/skills.

- [flight-search](../flight-search/SKILL.md) - searches for flights; pair its `merchant`/carrier codes with this skill to show airline names instead of raw codes.
