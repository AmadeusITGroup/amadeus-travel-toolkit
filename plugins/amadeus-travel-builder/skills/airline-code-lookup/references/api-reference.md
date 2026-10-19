# Airline code lookup - API reference

Detailed reference for the single `GET /v1/reference-data/airlines` endpoint. The base URL and authentication model (`X-API-Key` header from `AMADEUS_API_KEY`) are documented in [`../SKILL.md`](../SKILL.md#gateway-and-authentication).

## Endpoint

```
GET https://api.amadeus.com/v1/reference-data/airlines
```

Returns a list of airline records matching the requested IATA or ICAO codes.

## Query parameters

### `airlineCodes` *(optional)*

Comma-separated list of airline codes. Each entry is either:

- An **IATA** code (2 alphanumeric characters, e.g. `BA`, `AF`, `LH`, `2K`).
- An **ICAO** code (3 alphabetic characters, e.g. `BAW`, `AFR`, `DLH`).

You can mix IATA and ICAO codes in the same request:

```
?airlineCodes=BA,AFR,LH
```

Codes are case-insensitive (`ba` resolves the same as `BA`). The response always echoes them in canonical uppercase.

## Successful response (200)

```json
{
  "meta": {
    "count": 1,
    "links": {
      "self": "https://api.amadeus.com/v1/reference-data/airlines?airlineCodes=BA"
    }
  },
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

### `Airline` schema

| Field | Type | Notes |
|---|---|---|
| `type` | `"airline"` | Constant resource discriminator |
| `iataCode` | string | Always present for matched airlines |
| `icaoCode` | string \| undefined | Optional - placeholder records may omit it |
| `businessName` | string | Always present; uppercase |
| `commonName` | string \| undefined | Optional; uppercase; sometimes a short-form abbreviation |

### Behaviour for unmatched codes

The API does not return a 404 or an `errors` entry for unknown codes. Unmatched codes are simply absent from `data`. To detect misses:

```ts
const requested = ['BA', 'XX', 'LH']
const found = new Set(response.data.flatMap(a => [a.iataCode, a.icaoCode].filter(Boolean)))
const missing = requested.filter(code => !found.has(code.toUpperCase()))
```

## Error response (400)

| Code | Title |
|---|---|
| 572 | INVALID OPTION |

Returned when a code does not match the IATA or ICAO format. Example:

```json
{
  "errors": [
    {
      "status": 400,
      "code": 572,
      "title": "INVALID OPTION",
      "detail": "invalid query parameter format",
      "source": { "parameter": "airlineCodes", "example": "BA" }
    }
  ]
}
```

## Error response (429)

Too many requests against the gateway for your API key. Back off and retry with exponential backoff.

## Error response (500)

| Code | Title |
|---|---|
| 141 | SYSTEM ERROR HAS OCCURRED |

Generic upstream failure. Retry with backoff.

## Behaviour notes

- **`airlineCodes` is optional, but treat it as required.** Omitting it returns every airline in the reference database instead of erroring - a silent bulk fetch, not a per-lookup result.
- **Case-insensitivity.** `airlineCodes=ba` and `airlineCodes=BA` return identical results.
- **Mixed standards.** A single request can resolve a mix of IATA and ICAO codes - each is identified by its length (2 vs. 3 characters), not by a separate parameter.
- **Response order.** Not guaranteed to match request order. Always key by `iataCode` / `icaoCode`.
- **Batching.** Keep batches modest (≤ ~50 codes) for safe URL-length and predictable latency.
