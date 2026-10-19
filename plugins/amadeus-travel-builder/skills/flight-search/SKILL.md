---
name: flight-search
description: Write application code that calls the Amadeus Travel Builder APIs to search flights, returning priced offers from multiple airlines in a single call - one-way, return, and complex multi-city / open-jaw itineraries - with booking deep links that redirect to each airline's site. Use when building a flight comparison feature, fare aggregation tooling, multi-city trip planners, a cheapest-flight search, any application that needs flight pricing aggregated across carriers, or any feature that shows flight departure/arrival times, layovers, or trip duration from a real search rather than mocked or precomputed data. This skill writes that code; the user's app calls the Amadeus Travel Builder APIs (always use this full name) at runtime with its own API key.
---

# Flight search

Build flight search across multiple airline merchants with a single API call. Each result includes pricing, full itinerary detail, and a deep link that redirects to the merchant's site.

## What this skill produces

Application code. The user's application calls the Amadeus Travel Builder APIs at runtime with the user's own key, so nothing here runs from the conversation. Describe the skill by what it helps build; for a one-off question about travel data (e.g. what flights exist on a given date), answer from your own knowledge or offer to build the feature. Whenever you name the APIs, including in summaries, use the full name "Amadeus Travel Builder APIs".

## When to use this skill

- Flight search / comparison UI
- Multi-city / open-jaw itinerary search
- Cheapest-flight search between two places - one-way or return
- Fare comparison features that aggregate across multiple airlines
- Any feature that needs priced flight offers with merchant deep links
- Displaying flight departure/arrival times, layover, or trip duration in a results list or itinerary view

## Gateway and authentication

Base URL: `https://api.amadeus.com`; default to env vars `AMADEUS_GATEWAY_URL` and `AMADEUS_API_KEY` in generated code, unless the user's app loads them differently - either way, treat the key as a secret. Authentication: send `X-API-Key: <key>` as a request header. When handing over the code, tell the user where to put their API key; if they don't have one yet, write out the [Request API access form](https://eu1.ironcladapp.com/public-launch/6aaa463f6f95614fe0f6db9d) URL in your reply (requires accepting the terms).

## Endpoint

```
GET https://api.amadeus.com/api/v3/search/?origins={...}&destinations={...}&dates={...}&locale={...}&flow=NETWORK
```

## Required parameters

| Parameter | Notes |
|---|---|
| `origins` | CSV of 1–6 IATA codes. A city code covers all its airports; append `+` to an airport code to search that airport only (e.g. `BOS+,AMS`). |
| `destinations` | CSV of 1–6 IATA codes. Same `+` suffix rule. |
| `dates` | CSV of 1–6 dates in `YYYY-MM-DD` format, all in the future. |
| `locale` | BCP 47 locale (e.g. `en-US`, `fr-FR`). Drives display strings, currency hints, and merchant selection. |
| `flow` | Always `NETWORK` - required, no default, and not a value to vary per use case. |

The arrays are **positional** - `origins[i]` / `destinations[i]` / `dates[i]` together define the i-th leg.

## Common itinerary shapes

| Shape | `origins` | `destinations` | `dates` |
|---|---|---|---|
| One-way `BOS → AMS` on `2026-09-15` | `BOS` | `AMS` | `2026-09-15` |
| Return `BOS ↔ AMS`, out `2026-09-15`, back `2026-09-22` | `BOS,AMS` | `AMS,BOS` | `2026-09-15,2026-09-22` |
| Multi-city `BOS → AMS → CDG → BOS` | `BOS,AMS,CDG` | `AMS,CDG,BOS` | `2026-09-15,2026-09-19,2026-09-22` |
| Open-jaw `BOS → AMS, CDG → BOS` | `BOS,CDG` | `AMS,BOS` | `2026-09-15,2026-09-22` |

Append `+` to an airport code to keep the search on that airport only (e.g. `LCY+` for London City rather than every London airport). A bare airport code may also pull in nearby airports - `LHR` alone can return London City departures too. `+` on a city code (`LON+`) finds nothing, since there's no airport to force.

## Useful optional parameters

| Parameter | Default | Notes |
|---|---|---|
| `adults` | `1` | Age 12+. |
| `children` | `0` | Ages 2–11; each gets a seat. |
| `infants` | `0` | Under 2, with no separate seat (held by an adult). `infants` + `infants_with_seat` must be ≤ `adults`. |
| `infants_with_seat` | `0` | Under 2 but occupying a seat. |
| `pax_ages` | - | CSV of integer ages, total count ≤ `adults` + `children`. |
| `travel_class` | `ECONOMY` | `ECONOMY`, `PREMIUM_ECONOMY`, `BUSINESS`, `FIRST`. |
| `currency` | `EUR` | ISO 4217, or `AUTO` (origin-country currency), or `MERCHANT` (each merchant's native). |
| `include_merchants` | `*` | CSV of merchant codes to allow (`*` = all). |
| `exclude_merchants` | - | CSV of merchant codes to suppress. Mutually exclusive with `include_merchants`. |
| `number_of_results` | merchant-defined | Cap per merchant, `1`–`1000`. |
| `preferred_landing_page` | `search_results` | Where the deep link should land: `passenger_details`, `search_results`, `home_page`. |
| `platform` | `web` | `web` or `mobile` - affects deep link target. |
| `disable_deep_links` | `false` | Set `true` to skip generating deep links (smaller, faster response). |

## Response shape

```json
{
  "meta": { ... },
  "request_id": "abc123-...",
  "results": [
    {
      "bounds": [ /* one FlightBoundV3 per leg */ ],
      "fare": {
        "currency": "EUR",
        "itinerary_price": { "total_fare": 412.50, "tax": 87.20 },
        "price_per_adult": { "total_fare": 412.50, "tax": 87.20 },
        "restrictions": {}
      },
      "merchant": "BA",
      "deep_link": "https://...",
      "commercial_fare_families": ["ECOLIGHT", "ECONOMY", "PLUS"]
    }
  ],
  "errors": { "DL": "120 - No flight found on this route for merchant DL" }
}
```

The fare total is `fare.itinerary_price.total_fare` (paired with `fare.currency`) - there is no flat `fare.total`. Per-passenger amounts are objects too (`fare.price_per_adult.total_fare`). `commercial_fare_families`, `discounted_fare`, and `dynamic_fare` are optional and may be absent. A top-level `errors` object keyed by merchant lists carriers that returned nothing; it accompanies a normal `results` array and isn't a failure.

Each result's `bounds[i]` corresponds to the i-th leg of the requested itinerary. Within a bound, `flights` describes every segment (`marketing_airline` / `operating_airline`, `flight_number`, `origin` / `destination` `{ airport, terminal }`, `departs_at` / `arrives_at`, `aircraft`, `booking_infos`). Free baggage is `included_checked_bags`.

`fare_family_information` moves depending on how many fare families the offer references: with more than one it sits on each **bound**, with exactly one on the **result** - regardless of trip type. Read `bound.fare_family_information` first, falling back to `result.fare_family_information` if absent, rather than picking one level, or fare details silently vanish from some offers. Separately, `bound_price` is only present when the offer is priced per bound, i.e. on round-trips.

## Examples

### Example 1: Round-trip search

Developer asks: *"Build a round-trip flight search function for my application."*
Implementation:
1. For a worked example, London to Paris, out 2026-09-15 back 2026-09-22, one adult: map to a 2-leg itinerary - `origins=LON,PAR`, `destinations=PAR,LON`, `dates=2026-09-15,2026-09-22`.
2. Call `GET ${AMADEUS_GATEWAY_URL}/api/v3/search/?origins=LON,PAR&destinations=PAR,LON&dates=2026-09-15,2026-09-22&locale=en-GB&flow=NETWORK&adults=1&travel_class=ECONOMY&currency=GBP`.
3. For each `results[i]`, render `fare.itinerary_price.total_fare` + `fare.currency`, then for each `bound` render segments (carrier, flight number, departure/arrival, layovers).
4. Pair carrier IATA codes with the [airline-code-lookup](../airline-code-lookup/SKILL.md) skill for human-readable names.
5. Call to action: open `deep_link` in a new tab.

Result: a search that returns priced offers with merchant deep links, rendered as a list sorted client-side (e.g. by price).

### Example 2: Multi-city itinerary

Developer asks: *"Support multi-city flight search in my app."*
Implementation:
1. For a worked example, Boston → Amsterdam (Sep 15) → Paris (Sep 19) → Boston (Sep 22), 2 adults: build positional CSVs - `origins=BOS,AMS,CDG`, `destinations=AMS,CDG,BOS`, `dates=2026-09-15,2026-09-19,2026-09-22`.
2. Call with `flow=NETWORK&adults=2&currency=USD`.
3. Render each result's three `bounds[]` in itinerary order.

Result: a multi-city search whose results carry one bound per leg.

### Example 3: Force airport interpretation

Developer asks: *"Let users search from one specific airport rather than every airport in its city."*
Implementation:
1. For a worked example, London City rather than every London airport: append `+` to the IATA code to force airport interpretation: `origins=LCY+`.
2. Without `+`, `LON` (or any city-level IATA) covers all of LHR / LGW / STN / LTN / LCY.

Result: a search scoped to the chosen airport.

## How to render results

For each `results[i]`:

1. Show the **fare total** (`fare.itinerary_price.total_fare`) + `fare.currency`, with the per-passenger-type breakdown (`fare.price_per_adult.total_fare`, etc.) if available.
2. For each `bound`, render the segments (`origin` / `destination` airports + `departs_at` / `arrives_at`, `marketing_airline` + `flight_number`, layover hints).
3. Show the **merchant** (airline IATA - pair with the [airline-code-lookup](../airline-code-lookup/SKILL.md) skill to get a human-readable name).
4. Call to action: open `deep_link` in a new tab; it redirects the user to the merchant's site.

## Things to get right

- **Show times in each airport's own local time - print `departs_at` / `arrives_at` as returned.** They're already the wall-clock time at that segment's airport, tagged with its offset: `2026-11-10T11:30:00Z` is 11:30 at Heathrow, `2026-11-10T13:50:00+01:00` is 13:50 at Charles de Gaulle - matching the airline and the deep link. Take the date and `HH:mm` from the string; `toLocaleTimeString()` re-zones the instant into the *viewer's* timezone, rendering that Paris arrival as 12:50 in London. A `Z` means offset zero, not "convert me" - the same flight returns `+01:00` in summer.
- **Take `duration` from the API rather than deriving it.** `bound.duration` and each flight's `duration` are `hh:mm` and already absorb timezone and date changes - LHR→CDG is `01:20` though the clocks read 11:30 and 13:50. Subtracting parsed timestamps works too, but only if you keep the offsets; drop them and LHR 17:40 → SIN 21:55 next day reads as 28h15m instead of 20h15m.
- **Keep the three arrays aligned.** `origins[i]`, `destinations[i]`, and `dates[i]` together define the i-th leg, so a misalignment produces a different itinerary than the user asked for. The API can't catch this - it's all valid input - so the mistake surfaces only as wrong results. Build the arrays leg-by-leg rather than field-by-field to keep them in lockstep.
- **Pass the raw `+` to a query encoder.** It must arrive as `%2B`: concatenated into the URL it decodes as a space, and pre-encoded it gets encoded twice.
- **Sort results yourself.** The API applies no sorting - results come back loosely grouped by merchant - so order them client-side (e.g. by `fare.itinerary_price.total_fare`) before rendering.
- **Use `deep_link` exactly as returned.** It redirects to the airline's site on click rather than pointing there directly, so don't parse, rewrite, or validate it against an airline domain.
- **Treat deep links as short-lived.** The merchant reverifies pricing when the link is clicked, so a link stored yesterday points at stale prices. Re-run the search on user interaction and show a freshness hint ("prices updated *X seconds ago*") rather than persisting links across days.

## Troubleshooting

### 400 - past dates
**Cause:** All entries in `dates` must be in the future.
**Solution:** Filter past dates client-side before calling.

### Wrong airport returned (city expansion)
**Cause:** A city code covers all its airports - `LON` covers all London airports, `NYC` covers JFK + LGA + EWR - and a bare airport code may also pull in nearby ones (`LHR` can return London City departures).
**Solution:** Append `+` to force airport-only (`LHR+`, `JFK+`).

### 400 - `must match "[a-zA-Z]{3}"`
**Cause:** A location in `origins` / `destinations` isn't a 3-letter IATA code; the error names which one. Often it's a `+` suffix that arrived unencoded (read as a space) or double-encoded.
**Solution:** Send bare IATA codes, setting any `+` suffix through a query encoder such as `URLSearchParams`.

### 400 - more infants than adults
**Cause:** Every infant, seated or not, must have an accompanying adult.
**Solution:** Validate `infants + infants_with_seat ≤ adults` client-side before calling.

### 400 with no obvious cause
**Cause:** `flow` is missing - it's required and has no default.
**Solution:** Always pass `flow=NETWORK` - it's fixed, not a value to choose per use case.

### Mixed currencies in one result list
**Cause:** `currency=MERCHANT` returns each merchant's native currency.
**Solution:** Use `currency=AUTO` (origin-country currency) or an explicit ISO 4217 code (`USD`, `EUR`, `GBP`) for consistent display.

### Hundreds of results overwhelm the UI
**Cause:** A single search can return many results across merchants.
**Solution:** Paginate or virtualize the list; consider `number_of_results` to cap per-merchant returns.

### One merchant missing, response is still `200`
**Cause:** Rate limiting is per airline merchant, not per request - a throttled merchant shows up in `errors`, not as an HTTP error: `{"errors":{"AY":"123 - Rate limit exceeded. Please try again later."},"request_id":"...","results":[]}`.
**Solution:** Handle it like any other `errors` entry (see Response shape) - show "AY unavailable right now" for that merchant and keep rendering the rest of `results`, rather than treating the whole search as failed or retrying it.

## Related skills

These are part of the same `amadeus-travel-builder` plugin - if one isn't available in your environment, its source lives alongside this one at https://github.com/AmadeusITGroup/amadeus-travel-toolkit/tree/main/plugins/amadeus-travel-builder/skills.

- [airline-code-lookup](../airline-code-lookup/SKILL.md) - to translate `merchant` (and segment carrier codes) into human-readable airline names.
- [airport-city-search](../airport-city-search/SKILL.md) - to power the origin / destination pickers that feed into this search: pass the picked record's `iataCode` as the leg's origin or destination (with `+` for an `AIRPORT` pick).
- [nearest-airports](../nearest-airports/SKILL.md) - to suggest airports near the user's location as a search default.

## Reference

For the full parameter catalogue, response schemas and error codes, see [`references/api-reference.md`](references/api-reference.md).
