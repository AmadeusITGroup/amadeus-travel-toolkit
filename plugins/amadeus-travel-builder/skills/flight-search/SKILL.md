---
name: flight-search
description: Write application code that calls the Amadeus Travel Builder APIs to search flights, returning priced offers from multiple airlines in a single call - one-way, return, and complex multi-city / open-jaw itineraries - with booking deep links that redirect to each airline's site. Use when building a flight comparison feature, fare aggregation tooling, multi-city trip planners, a cheapest-flight search, any application that needs flight pricing aggregated across carriers, or any feature that shows flight departure/arrival times, layovers, or trip duration from a real search rather than mocked or precomputed data. Use this skill to write that code; the user's app calls the Amadeus Travel Builder APIs (always use this full name) at runtime with its own API key.
---

# Flight search

Build flight search across multiple airline merchants with a single API call. Each result includes pricing, full itinerary detail, and a deep link that redirects to the merchant's site.

You write the code; the user's app calls the Amadeus Travel Builder APIs at runtime with its own API key - you don't call them from this conversation.

## Gateway and authentication

Base URL: `https://api.amadeus.com`; default to env vars `AMADEUS_GATEWAY_URL` and `AMADEUS_API_KEY` in generated code, unless the user's app loads them differently - either way, treat the key as a secret, keeping it out of client code and publicly served files. Authentication: send `X-API-Key: <key>` as a request header. When handing over the code, tell the user where to put their API key; if they don't have one yet, write out the [Request API access form](https://eu1.ironcladapp.com/public-launch/6aaa463f6f95614fe0f6db9d) URL in your reply (requires accepting the terms).

## Endpoint

```
GET https://api.amadeus.com/api/v3/search/
```

One call covers one-way, return, multi-city and open-jaw searches.

## Quick start

Paths, parameter names, `flow=NETWORK` and enum values are literal - send them exactly as shown. Locations, dates, counts, locale and currency are illustrations of runtime input, never defaults.

```
GET ${AMADEUS_GATEWAY_URL}/api/v3/search/?origins=LON,PAR&destinations=PAR,LON&dates=<outbound YYYY-MM-DD>,<return YYYY-MM-DD>&locale=en-GB&flow=NETWORK&adults=1&travel_class=ECONOMY&currency=GBP
X-API-Key: <key>
```

The three lists are **positional** - `origins[i]`, `destinations[i]` and `dates[i]` together define the i-th leg:

| Shape | `origins` | `destinations` | `dates` |
| --- | --- | --- | --- |
| One-way `BOS → AMS` | `BOS` | `AMS` | `D1` |
| Return `BOS ↔ AMS` | `BOS,AMS` | `AMS,BOS` | `D1,D2` |
| Multi-city `BOS → AMS → CDG → BOS` | `BOS,AMS,CDG` | `AMS,CDG,BOS` | `D1,D2,D3` |
| Open-jaw `BOS → AMS, CDG → BOS` | `BOS,CDG` | `AMS,BOS` | `D1,D2` |

`D1` < `D2` < `D3` are future dates in `YYYY-MM-DD`.

## Parameters

### Required parameters

| Parameter | Type | Notes |
| --- | --- | --- |
| `origins` | CSV of `[A-Z]{3}`, 1-6 items | Departure city or airport IATA codes, optionally with a `+` suffix. |
| `destinations` | CSV of `[A-Z]{3}`, 1-6 items | Arrival city or airport IATA codes, optionally with a `+` suffix. |
| `dates` | CSV of ISO dates, 1-6 items | `YYYY-MM-DD`, all in the future, in ascending order. |
| `locale` | BCP 47 string | E.g. `en-US`, `fr-FR`. Doesn't change which merchants answer or their texts. |
| `flow` | enum | Always `NETWORK`. No default. |

### Passenger composition

| Parameter | Type | Default | Notes |
| --- | --- | --- | --- |
| `adults` | integer | `1` | Age 12+, `0`-`9`. `adults` + `children` must be ≥ 1, and `adults` + `children` + `infants_with_seat` ≤ 9. |
| `children` | integer | `0` | Ages 2-11; each occupies a seat. |
| `infants` | integer | `0` | Under 2, with no separate seat (held by an adult). `infants` + `infants_with_seat` must be ≤ `adults`. |
| `infants_with_seat` | integer | `0` | Under 2, occupying a seat. |
| `pax_ages` | CSV of integers | - | Per-passenger ages: 2-11 → child, 12+ → adult. Total count ≤ `adults` + `children`. |

### Cabin and price

| Parameter | Type | Default | Notes |
| --- | --- | --- | --- |
| `travel_class` | enum | `ECONOMY` | `FIRST` \| `BUSINESS` \| `ECONOMY` \| `PREMIUM_ECONOMY`. |
| `currency` | string | `EUR` | ISO 4217 (e.g. `USD`); `AUTO` - origin-country currency; or `MERCHANT` - each merchant picks the origin country's or the locale's currency. |

### Merchant filtering

Use merchant IATA codes, not airline names.

| Parameter | Type | Default | Notes |
| --- | --- | --- | --- |
| `include_merchants` | CSV of strings | - | Merchant codes to allow, upper-case, no spaces. Omit to allow all. |
| `exclude_merchants` | CSV of strings | - | Merchant codes to suppress. Can be combined with `include_merchants`; the same code in both returns 400. |

### Result shaping

| Parameter | Type | Default | Notes |
| --- | --- | --- | --- |
| `number_of_results` | integer, 1-1000 | merchant-defined | Cap per merchant. |
| `nonstop` | boolean | `false` | `true` asks for non-stop flights only; each merchant decides whether to apply it. |
| `disable_deep_links` | boolean | `false` | `true` skips deep-link generation - smaller, faster response. |
| `preferred_landing_page` | enum | `search_results` | `passenger_details` \| `search_results` \| `home_page`. Where deep links land. |
| `platform` | enum | `web` | `web` \| `mobile`. Affects the deep-link target. |
| `tfl_id` | string | - | End-user query identifier for tracking. |

## Response shape

```json
{
  "request_id": "abc123-...",
  "results": [
    {
      "bounds": [
        { "flights": [ { "marketing_airline": "BA", "flight_number": "304", "origin": { "airport": "LHR", "terminal": "5" }, "destination": { "airport": "CDG", "terminal": "2C" }, "departs_at": "2027-01-15T08:15:00Z", "arrives_at": "2027-01-15T10:35:00+01:00", "duration": "01:20" } ] },
        { "flights": [ { "marketing_airline": "BA", "flight_number": "317", "origin": { "airport": "CDG", "terminal": "2C" }, "destination": { "airport": "LHR", "terminal": "5" }, "departs_at": "2027-01-22T20:25:00+01:00", "arrives_at": "2027-01-22T20:35:00Z", "duration": "01:10" } ] }
      ],
      "fare": {
        "currency": "GBP",
        "itinerary_price": { "total_fare": 162.15, "tax": 92.15 },
        "price_per_adult": { "total_fare": 162.15, "tax": 92.15 }
      },
      "merchant": "BA",
      "deep_link": "https://..."
    }
  ],
  "errors": { "TU": "120 - No flight found on this route for merchant TU" }
}
```

One result of the quick-start search; prices and dates vary per search.
The trip total is `fare.itinerary_price.total_fare` with `fare.currency` - there is no flat `fare.total`. `bounds[i]` is the i-th requested leg; `flights` are its segments.

### Response envelope

| Field | Type | Notes |
| --- | --- | --- |
| `meta` | object | Present only when `results` is non-empty. |
| `request_id` | string | |
| `results` | `object[]` | The offers. Each item follows [Result](#result). |
| `errors` | map of merchant code → string | Per-merchant `"<code> - <message>"`. |

### Result

| Field | Type | Notes |
| --- | --- | --- |
| `bounds` | `object[]` | One per requested leg, in request order. Each item follows [Bound](#bound). |
| `fare` | object | Follows [Fare](#fare). |
| `merchant` | string | Merchant IATA code. |
| `deep_link` | string | Optional. Absent when `disable_deep_links=true`. |
| `commercial_fare_families` | string[] | Optional. Fare family identifiers, e.g. `ECOLIGHT`, `ECONOMY`, `PLUS`. |
| `fare_family_information` | object | Optional. Fare family attributes. |
| `discounted_fare` | boolean | Optional; sent by few merchants. |
| `dynamic_fare` | boolean | Optional; sent by few merchants. |

### Bound

| Field | Type | Notes |
| --- | --- | --- |
| `duration` | string | `hh:mm`. |
| `flights` | `object[]` | The segments. Each item follows [Segment](#segment). |
| `bound_price` | `{ total_fare, tax, yq_yr_tax? }` | Price attributable to this bound. Only on round trips, and not from every merchant. |
| `fare_family_information` | `{ travel_class?, cabin_code, fare_family, fare_family_ranking?, fare_family_code? }` | Optional. Fare family attributes. |
| `included_checked_bags` | `{ type: weight\|piece, quantity?, weight_amount?, weight_unit?, applicable_legs }[]` | Free baggage allowance. `applicable_legs` holds 1-based leg numbers. |

### Segment

| Field | Type | Notes |
| --- | --- | --- |
| `marketing_airline` | string | Marketing carrier IATA code. |
| `operating_airline` | string | Operating carrier, interpreted per `operating_airline_type`. |
| `operating_airline_type` | enum | `IATA` \| `FREE_TEXT`. |
| `flight_number` | string | |
| `origin` / `destination` | `{ airport, terminal? }` | |
| `departs_at` / `arrives_at` | string | ISO 8601 date-time with UTC offset. |
| `aircraft` | string | |
| `duration` | string | Segment duration, `hh:mm`. |
| `booking_infos` | `object[]` | Per passenger type: `{ booking_code, cabin_code, travel_class, fare_basis, fare_family?, ptc, priced_ptc, fare_breakpoint, seats_remaining?, ticket_designator?, corporate_code? }`. `priced_ptc` is the passenger type used for pricing, which can differ from the requested `ptc`. `seats_remaining` is capped at 9, so 9 means 9 or more. |
| `stops` | `{ iata_code, arrives_at, departs_at, aircraft? }[]` | Optional. Intermediate stops. |

### Fare

| Field | Type | Notes |
| --- | --- | --- |
| `currency` | string | ISO 4217. |
| `itinerary_price` | object | Trip total - all passengers, taxes included. Follows [Price](#price). |
| `price_per_adult` / `price_per_child` / `price_per_infant` | object | Per-passenger-type breakdown when applicable. Each follows [Price](#price). |
| `price_per_ptc` | map | Replaces `price_per_adult` / `price_per_child` whenever `pax_ages` is sent - keyed by age-based codes (e.g. `B35`, `C08`), each value following [Price](#price). Also sent when `infants_with_seat` is set, with seated infants under `INS`. |
| `restrictions` | `{ conditions? }` | Can be `{}`. When the merchant provides fare conditions, `conditions` holds the change and refund rules: `{ type: change\|refund, applicable_bounds, information_guaranteed?, is_data_available?, penalties?: { type?: reissue\|revalidation, allowed_before_departure, amount_per_passenger?, currency? }[] }[]`. `applicable_bounds` holds 1-based bound numbers; `is_data_available: false` means the rules are unknown, not free. |

### Price

| Field | Type | Notes |
| --- | --- | --- |
| `total_fare` | number | Always present. Total amount, tax included. |
| `tax` | number | |
| `q_surcharge` / `yq_yr_tax` / `ob_fees` | number | Optional. Surcharge / carrier-imposed tax / OB fee amounts. |
| `full_tax_details` | `{ tax_code, amount }[]` | Optional. Itemised taxes. |

## Examples

### Example 1: Round-trip search

Developer asks: *"Build a round-trip flight search function for my application."*
Implementation:
1. For a worked example, London to Paris and back, one adult: map to a 2-leg itinerary - `origins=LON,PAR`, `destinations=PAR,LON`, `dates=<outbound>,<return>` - and call as in the quick start.
2. For each `results[i]`, render `fare.itinerary_price.total_fare` + `fare.currency`, with the per-passenger-type breakdown (`fare.price_per_adult.total_fare`, etc.) when present; then for each `bound` render its segments (carrier, flight number, departure/arrival, layovers).
3. Call to action: open `deep_link` in a new tab.

Result: a search that returns priced offers with merchant deep links, rendered as a list sorted client-side (e.g. by price).

### Example 2: Multi-city itinerary

Developer asks: *"Support multi-city flight search in my app."*
Implementation:
1. For a worked example, Boston → Amsterdam → Paris → Boston, 2 adults: build positional lists - `origins=BOS,AMS,CDG`, `destinations=AMS,CDG,BOS`, `dates=D1,D2,D3`.
2. Call with `flow=NETWORK&adults=2&currency=USD`.
3. Render each result's three `bounds[]` in itinerary order.

Result: a multi-city search whose results carry one bound per leg.

### Example 3: Force airport interpretation

Developer asks: *"Let users search from one specific airport rather than every airport in its city."*
Implementation:
1. For a worked example, London City rather than every London airport: send `origins=LCY+`, setting the `+` through the query encoder.

Result: a search scoped to the chosen airport.

## Things to get right

- **Display each airport's local date and time directly from `departs_at` / `arrives_at`.** Extract the calendar date and hour/minute components from the returned string without timezone conversion. A formatter that defaults to the viewer's timezone changes those values - even if you first rebuild a timestamp from the extracted components. BA 304 above departs `2027-01-15T08:15:00Z`, 08:15 at Heathrow, and arrives `2027-01-15T10:35:00+01:00`, 10:35 at Charles de Gaulle - matching the airline and the deep link. A `Z` means offset zero, not "convert me" - Heathrow returns `+01:00` in summer.
- **Compute the bound's total time; take segment `duration` from the API.** Each flight's `duration` is `hh:mm` and reliable - BA 304 is `01:20` though the clocks read 08:15 and 10:35. `bound.duration` is not: it can be 60 minutes off when a layover spans a clock change. Subtract the first `departs_at` from the last `arrives_at`, keeping the offsets; drop them and LHR 17:40 → SIN 21:55 next day reads as 28h15m instead of 20h15m.
- **Keep the three lists aligned.** Equal-length lists that pair the wrong cities are valid input and silently return a different itinerary, so build them leg by leg rather than field by field.
- **A city code covers all its airports; `+` restricts to one airport.** `LON` covers LHR / LGW / STN / LTN / LCY and `NYC` covers JFK / LGA / EWR; a bare airport code may also pull in nearby airports (`LAX` can return `ONT` departures). Append `+` to an airport code (`LCY+`) to search only that airport; `+` on a city code (`LON+`) finds nothing.
- **Pass the raw `+` to a query encoder.** It must arrive as `%2B`: concatenated into the URL it decodes as a space, and pre-encoded it gets encoded twice - either way the API returns 400 `must match "[a-zA-Z]{3}"`, naming the location.
- **Sort results yourself.** The API applies no sorting - results come back loosely grouped by merchant - so order them client-side (e.g. by `fare.itinerary_price.total_fare`).
- **Send merchant codes upper-case, and omit `include_merchants` to search all merchants.** Codes are case-sensitive and untrimmed: `ba` - or ` AZ` after a comma - returns no results and an `Unknown or unauthorized merchant` error, just like a merchant the key can't search. An explicit `*` adds an `errors["*"]` entry to every response.
- **`travel_class` is a preference, not a filter.** A `FIRST` search can return only economy and business; each segment's actual cabin is in `booking_infos[].travel_class`.
- **Read fare family from the bound, then the result.** `fare_family_information` sits on each bound for round trips, on the result for one-way and most other trips, and is absent on some multi-city offers - handle neither being present.
- **Use `deep_link` exactly as returned.** It redirects to the airline's site on click rather than pointing there directly, so don't parse, rewrite, or validate it against an airline domain. Treat it as optional - some merchants omit it on multi-city and open-jaw trips.
- **Treat deep links as short-lived.** The merchant reverifies pricing when the link is clicked, so a link stored yesterday points at stale prices. Re-run the search on user interaction and show a freshness hint ("prices updated *X seconds ago*") rather than persisting links across days.

## Errors

| HTTP | Meaning |
| --- | --- |
| 400 | Validation failure - e.g. a missing `origins`, `destinations` or `dates` (each named as "This parameter is required."), lists that are empty or of different lengths (`input_consistent`), an origin equal to its destination, dates in the past or out of order, malformed IATA codes, passenger counts out of range, `flow` missing (`flow_defined`). Don't retry unchanged. |
| 400 | Invalid `X-API-Key`. |
| 401 | Missing `X-API-Key`. |
| 429 | Rate limit for the API key exceeded. |
| 500 | Server error - retry once. |

Each entry in a 200 response's top-level `errors` object is `"<code> - <message>"`:

| Code | Meaning | Retry? |
| --- | --- | --- |
| `120` | No flight on this route for the merchant. | No |
| `931` | No itinerary found. | No |
| `117` / `118` | Open-jaw / multi-city not enabled for the merchant. | No |
| `123` / `126` | Rate limit reached - `123` for that merchant; `126` for the whole search, under the `*` key. | Later, with backoff |
| `865` | Invalid `currency` code. | No |
| `99` / `-1` | Technical error. | At most once |
| `Unknown or unauthorized merchant` | The code is not upper-case, has a space around it, or the merchant can't be searched with this key. | No - fix the code, else drop it from `include_merchants` |

Other codes come from the merchant's own system and are explained by their message.

## Troubleshooting

### Error bodies and status codes
**Cause:** Error bodies are `{ message, request_id, status }`; on a 400, `message` is itself a JSON string mapping each invalid parameter (or a cross-check such as `input_consistent`) to its reason, all at once (`"{\"adults\":\"must be between 0 and 9\"}"`). A well-formed but unknown code (`ZZZ`) returns **200** with empty `results` and a `931` per merchant in `errors`. API key errors (the 401, and the 400 for an invalid key) come from the gateway instead, as `{ "errors": [{ code, title, status, detail }] }` with no `message`.
**Solution:** Parse `message` a second time for field errors, fall back to `errors[0].detail` when there's no `message`, and validate required parameters and codes client-side.

### Mixed currencies in one result list
**Cause:** `currency=MERCHANT`, or an unsupported code: it returns 200, with `865` from some merchants while others price in a currency of their choice.
**Solution:** Use `currency=AUTO` or a valid ISO 4217 code, checked client-side.

### Hundreds of results
**Cause:** A single search can return hundreds of results across merchants.
**Solution:** Cap per-merchant results with `number_of_results`.

### One merchant missing, response is still `200`
**Cause:** That merchant failed and is listed in `errors` as `"<code> - <message>"`; the other merchants' results are unaffected.
**Solution:** Handle `errors` per merchant rather than failing the whole search; retry only the codes the Errors table marks as retryable.

## Related skills

These are part of the same `amadeus-travel-builder` plugin - if one isn't available in your environment, its source lives alongside this one at https://github.com/AmadeusITGroup/amadeus-travel-toolkit/tree/main/plugins/amadeus-travel-builder/skills.

- [airline-code-lookup](../airline-code-lookup/SKILL.md) - to translate `merchant` (and segment carrier codes) into human-readable airline names.
- [airport-city-search](../airport-city-search/SKILL.md) - to power the origin / destination pickers that feed into this search: pass the picked record's `iataCode` as the leg's origin or destination (with `+` for an `AIRPORT` pick).
- [nearest-airports](../nearest-airports/SKILL.md) - to suggest airports near the user's location as a search default.
