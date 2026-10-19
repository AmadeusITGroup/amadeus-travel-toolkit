# Flight search - API reference

Detailed reference for `GET /api/v3/search/`. The base URL and authentication model (`X-API-Key` header from `AMADEUS_API_KEY`) are documented in [`../SKILL.md`](../SKILL.md#gateway-and-authentication).

## Endpoint

```
GET https://api.amadeus.com/api/v3/search/
```

## Query parameters

### Required

| Parameter | Type | Notes |
|---|---|---|
| `origins` | CSV `[A-Z]{3}`, `1`–`6` items | IATA codes for departure cities/airports. Append `+` to force airport interpretation (`BOS+`, `LCY+`). |
| `destinations` | CSV `[A-Z]{3}`, `1`–`6` items | IATA codes for arrival cities/airports. Same `+` rule. |
| `dates` | CSV ISO dates, `1`–`6` items | `YYYY-MM-DD`, all in the future. |
| `locale` | BCP 47 string | E.g. `en-US`, `fr-FR`. Drives display strings and currency hints. |
| `flow` | enum | Always `NETWORK`. No default, required on every call - not a value to vary per use case. |

The arrays are **positional**: `origins[i]` / `destinations[i]` / `dates[i]` together define the i-th leg of the itinerary.

### Passenger composition

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `adults` | integer | `1` | Age 12+. |
| `children` | integer | `0` | Ages 2–11; each occupies a seat. |
| `infants` | integer | `0` | Under 2, with no separate seat (held by an adult). `infants` + `infants_with_seat` must be ≤ `adults`. |
| `infants_with_seat` | integer | `0` | Under 2, occupying a seat. |
| `pax_ages` | CSV integers | - | Per-passenger ages. 2–11 → child, 12+ → adult. Total count ≤ `adults` + `children`. |

### Cabin and price

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `travel_class` | enum | `ECONOMY` | `FIRST` \| `BUSINESS` \| `ECONOMY` \| `PREMIUM_ECONOMY`. Result will include the specified class **or higher**. |
| `currency` | string | `EUR` | ISO 4217 (e.g. `USD`), or `AUTO` (origin-country currency), or `MERCHANT` (each merchant's native currency). |

### Merchant filtering

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `include_merchants` | CSV strings | `*` | Merchant codes to allow. `*` allows all. |
| `exclude_merchants` | CSV strings | - | Merchant codes to suppress. Cannot combine with `include_merchants`. |
| `merchant_tokens` | CSV strings | - | Tokens granting access to specific merchant content. |

### Result shaping

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `number_of_results` | integer (1–1000) | merchant-defined | Cap per merchant. |
| `disable_deep_links` | boolean | `false` | Skip deep-link generation in the response. |
| `preferred_landing_page` | enum | `search_results` | `passenger_details` \| `search_results` \| `home_page`. Where deep links should land. |
| `platform` | enum | `web` | `web` \| `mobile` - affects deep link target. |
| `tfl_id` | string | - | End-user query identifier for tracking. |

## Successful response (200) - `SingleDateFlightsSearchResponseV3`

```json
{
  "meta": { ... },
  "request_id": "abc123-...",
  "results": [ /* SingleDatePriceResultV3[] */ ],
  "errors": {
    "DL": "120 - No flight found on this route for merchant DL",
    "QR": "866 - ... fares do not seem to exist for the selected itinerary ..."
  }
}
```

A successful response includes `meta`, `request_id` and `results`, plus a top-level **`errors`** object keyed by merchant code - per-merchant "no content / no flight" messages for merchants that returned nothing. It is informational; a populated `results` array alongside it is normal. Don't treat the presence of `errors` as a failed search.

### `SingleDatePriceResultV3`

| Field | Type | Notes |
|---|---|---|
| `bounds` | `FlightBoundV3[]` | One per leg in the itinerary, in the same order as the request. |
| `fare` | `FareV3` | Itinerary total + per-passenger-type breakdown (each a `PriceDetails` object), fees, currency, restrictions. |
| `merchant` | string | Merchant IATA code. |
| `deep_link` | string | Redirects to the merchant's site on click; use as-is. Absent when `disable_deep_links=true`. |
| `commercial_fare_families` | string[] | Fare family identifiers (e.g. `ECOLIGHT`, `ECONOMY`, `PLUS`). |
| `fare_family_information` | object | Attributes of the fare (baggage, refundability, etc.). Present here when the offer references exactly one fare family; on each bound instead when it references more than one - see `FlightBoundV3`. |
| `discounted_fare` | `DiscountedFare` \| `null` | Present only when a negotiated discount applies to the offer. |
| `dynamic_fare` | `DynamicFare` \| `null` | Present only when a promotional discount applies to the offer. |

### `FlightBoundV3`

A bound describes one leg. Fields:

| Field | Type | Notes |
|---|---|---|
| `duration` | string | Bound duration, `hh:mm`. |
| `flights` | `FlightDetailsV3[]` | The segments - see below. |
| `bound_price` | `BoundPriceDetails` | `{ total_fare, tax, original_fare?, yq_yr_tax? }` - price attributable to this bound. Only present when the offer is priced per bound, i.e. on round-trips; absent on one-way and multi-city. |
| `fare_family_information` | `FareFamilyInformation` | `{ travel_class, cabin_code, fare_family, fare_family_ranking, fare_family_code }`. Present here when the offer references more than one fare family; on the result instead when it references one, regardless of trip type - so read `bound.fare_family_information` first, falling back to `result.fare_family_information` if absent. |
| `included_checked_bags` | `BaggageAllowanceDetail[]` | Free allowance: `{ type: weight\|piece, quantity?, weight_amount?, weight_unit? }`. |
| `additional_chargeable_bags` | `AdditionalChargeableBaggage[]` | Bags not included in the fare: `{ description, price, original_price?, currency, passengerType, applicable_legs }`. |

#### `FlightDetailsV3` (each segment)

| Field | Type | Notes |
|---|---|---|
| `marketing_airline` | string | Marketing carrier IATA. |
| `operating_airline` | string | Operating carrier - interpret per `operating_airline_type`. |
| `operating_airline_type` | enum | `IATA` \| `FREE_TEXT`. |
| `flight_number` | string | |
| `origin` / `destination` | `AirportTerminal` | `{ airport, terminal? }`. |
| `departs_at` / `arrives_at` | string | ISO 8601 date-time, always offset-qualified, in **that segment's own airport's local time** - e.g. `2026-11-11T15:10:00+09:00` for an ICN arrival. An airport at UTC+0 serialises with `Z`; that's an offset of zero, not a UTC-normalised value. Display verbatim; parse only for arithmetic. |
| `aircraft` | string | |
| `duration` | string | Segment duration, `hh:mm`. |
| `booking_infos` | `BookingInfoV3[]` | Per-PTC: `{ booking_code, cabin_code, travel_class, fare_basis, fare_family, ptc, seats_remaining, ticket_designator }`. |
| `stops` | `FlightStop[]` | Technical stops, if any. |

### `FareV3`

| Field | Type | Notes |
|---|---|---|
| `currency` | string | ISO 4217. |
| `itinerary_price` | `PriceDetails` | Total price of the trip (all passengers, taxes included). See `PriceDetails`. |
| `price_per_adult` / `price_per_child` / `price_per_infant` | `PriceDetails` | Per-passenger-type breakdown when applicable. |
| `price_per_ptc` | map | Present only when `pax_ages` is sent and prices don't map to standard ADT/CH - keyed by PTC code, each a `PriceDetails`. |
| `fees` | `Fees` | `{ creditcard_fees, service_fees }` - maps keyed by card type / fee type. |
| `restrictions` | `Restrictions` | Refundability / change-penalty flags (may be empty `{}`). |

#### `PriceDetails`

| Field | Type | Notes |
|---|---|---|
| `total_fare` | number | **Required.** Total amount, tax included. |
| `tax` | number | Tax amount. |
| `original_fare` | number | Amount before any discount, tax included. |
| `q_surcharge` / `yq_yr_tax` / `ob_fees` | number | Surcharge / carrier-imposed tax / OB fee amounts. |
| `full_tax_details` | `FullTaxDetails[]` | Itemised taxes. |

So the itinerary total is `fare.itinerary_price.total_fare` (+ `fare.currency`), **not** a flat `fare.total`. Per-adult is `fare.price_per_adult.total_fare`.

## Errors

| HTTP | Meaning |
|---|---|
| 400 | Bad request - validation failure. Common causes: misaligned `origins`/`destinations`/`dates` arrays (the API rejects lists of differing sizes), past dates, invalid IATA codes, `infants` + `infants_with_seat` > `adults`. |

Per-merchant failures don't fail the whole search: merchants that returned nothing are reported in the top-level **`errors`** object (keyed by merchant code) alongside a populated `results` array. Log them if useful, but a search with both `errors` and `results` is a success.

Rate limiting is per merchant, not per request, so it surfaces the same way - a throttled merchant appears in `errors` on an otherwise-`200` response, e.g. `{"errors":{"AY":"123 - Rate limit exceeded. Please try again later."},"results":[]}`.

## Behaviour notes

- **Positional arrays.** Lists of differing lengths are rejected with a 400. Equal-length lists that pair the wrong cities are perfectly valid input, so a misalignment is accepted and silently returns a different itinerary - keep them aligned.
- **City vs airport interpretation.** A city code covers all its airports: `LON` covers LHR / LGW / STN / LTN / LCY; `NYC` covers JFK / LGA / EWR. A bare airport code may also pull in nearby airports (`LHR` can return LCY departures); `+` restricts it to that airport. `+` on a city code (`LON+`) returns no results.
- **Currency modes.** Three are supported: an explicit ISO 4217 code, `AUTO` (origin-country currency for consistent display), `MERCHANT` (each merchant's native currency, leading to mixed currencies in the result set).
- **`travel_class` is "or higher"** - searching `ECONOMY` may return premium-economy or business results too if they are price-competitive.
- **Result cardinality.** Hundreds of results across merchants are common - paginate / virtualize.
- **No sorting.** Results come back loosely grouped by merchant, not ordered by price or duration - sort client-side.
- **Deep link freshness.** Treat as time-sensitive; don't cache across days. Re-run the search on user click rather than relying on a stored deep link.
