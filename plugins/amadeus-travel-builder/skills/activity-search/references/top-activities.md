# Activity search - top activities

`GET https://api.amadeus.com/recommendation/v1/top-activities` returns the top-performing products at a destination, as a JSON array of products (schema in `product-schema.md`).

## Query parameters

At least one locator is required: `lat`+`lon`, `iataCode` or `cityName`.

| Parameter | Type | Notes |
| --- | --- | --- |
| `iataCode` | string | City or airport IATA code, e.g. `PAR`, `LON`, `BCN`. |
| `cityName` | string | English city name, e.g. `Madrid`, `Vienna`. |
| `lat` | double | Latitude. Pair with `lon`. |
| `lon` | double | Longitude. Pair with `lat`. |
| `radius` | double | Kilometres around the coordinates, or around the city centre for `cityName`/`iataCode`. Greater than `0`. Optional. |
| `maxRecommendations` | int32 | `1`-`10`, default `10`. |
| `computationWeight` | string | Scoring weights as a JSON string. Fields: `title`, `price`, `geolocation`, `tripContext`, `travelerType`, `interest`, `rating`, `season`, `engagement` - all optional doubles, higher means more weight. |
| `minContentScore` | float | Greater than `0`, up to `1`. Doesn't change the results. |
| `currency` | string | ISO 4217, e.g. `EUR`, `USD`. Default `EUR`. |

## Response

`200 OK` with a JSON array of products.

No pagination: `maxRecommendations` caps the result count.

## Errors

| HTTP | Code | Meaning |
| --- | --- | --- |
| 400 | 6025 | Unknown destination, a non-English `cityName` (`Wien`), or coordinates with no products around them. |
| 400 | 6043 | Invalid parameter value or no locator, e.g. `maxRecommendations` above 10 or `radius=0`. Unknown query parameters are ignored. |
| 500 | 2004 | Unsupported `currency`. Not transient - don't retry. |
