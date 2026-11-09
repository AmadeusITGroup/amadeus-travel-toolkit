# Activity search - product schema

Fields of the product objects all three endpoints return; endpoint differences are noted per field.

## Contents

- Identity fields (id, productId, externalId)
- Text and media fields (title, description, experienceHighlights, galleryImages, thumbnailImage, languages, aiGenerated)
- Price fields (minPrice, maxPrice, priceCurrency, pricingUnit)
- Booking fields (bookingUrl, travelerUrl, bookingEngine, bookingEngineName, onlineBookable, instantConfirmation, instantDelivery, redemptionMethod, skipTheLine)
- Location fields (address, geoLocation, cityName, countryName, zipCode, continentCode)
- Schedule and duration fields (duration, durationHours, availabilityType, availabilityPeriods, itinerary)
- Policy and inclusion fields (isCancellable, cancellationCutOff*, modalities*, additionalInformation, handiAccess)
- Rating and ranking fields (rating, numberOfRatings, bestseller, relevanceScore, descriptors)
- Classification fields (categories, taxonomies) and their item fields
- Supplier and record fields (supplier, creationDate, updateDate)

## Identity fields

| Field | Type | Notes |
| --- | --- | --- |
| `id` | int64 | Encodes the search context and changes on every search call. On get product it equals `productId`, since there's no search context. If retained, decode it losslessly at parse time (string or 64-bit integer): it exceeds 53-bit float precision, so wrapping it in `String()` after a normal `JSON.parse` / `response.json()` is too late. |
| `productId` | int64 | Static catalogue ID. Safe to parse as a normal JSON number. |
| `externalId` | string | Booking-engine-specific ID. |

## Text and media fields

| Field | Type | Notes |
| --- | --- | --- |
| `title` | string | Display title. |
| `description` | string | Full description. |
| `experienceHighlights` | string | Free text, bullet-list-like. |
| `galleryImages` | `{ url, order }[]` | Displayable photos. |
| `thumbnailImage` | string | Image ID. |
| `availableLanguages` | string[] | Language codes for the product's available translations. |
| `displayedLanguage` | string | Reported response language code. |
| `guideLanguages` | `{ live?: string[] }` | Languages a live guide is offered in. |
| `aiGenerated` | `{ translation?: boolean }` | Which parts are machine-generated, e.g. `{ "translation": true }` for machine-translated `title`/`description`. |

## Price fields

| Field | Type | Notes |
| --- | --- | --- |
| `minPrice` / `maxPrice` | double | |
| `priceCurrency` | string | ISO 4217. On get product, the product's stored currency. |
| `pricingUnit` | string | Always `NONE`. |

## Booking fields

| Field | Type | Notes |
| --- | --- | --- |
| `bookingUrl` | string | Where to send the user to book. |
| `travelerUrl` | string | Tracking-wrapped link to the traveler-facing product page; distinct from `bookingUrl`. |
| `bookingEngine` | string | `HOLIBOB_VIATOR`, `HOLIBOB_TIQETS`, `HOLIBOB_MUSEMENT` or `HOLIBOB`. |
| `bookingEngineName` | string | Display name, e.g. `Holibob`. |
| `onlineBookable` | boolean | |
| `instantConfirmation` | boolean | |
| `instantDelivery` | boolean | |
| `redemptionMethod` | string | E.g. `DIGITAL`. |
| `skipTheLine` | boolean | |

## Location fields

| Field | Type | Notes |
| --- | --- | --- |
| `address` | string | Free-text address. |
| `geoLocation` | `{ latitude, longitude, geoHash? }` | |
| `cityName` / `countryName` | string | |
| `zipCode` | string | Postal code, e.g. `"78000"`; present only when the venue has one. |
| `continentCode` | string | E.g. `"EU"`. Returned only by recommended products - never by top activities or get product, even for the same product. |

## Schedule and duration fields

| Field | Type | Notes |
| --- | --- | --- |
| `duration` | string | Free text, e.g. "2 hours". Can be empty. |
| `durationHours` | double | Numeric duration when known. |
| `availabilityType` | string | E.g. `OPENING_HOURS`. |
| `availabilityPeriods` | `{ weeklyTimes?, weeklyPeriods? }` | Opening hours; empty on most products. `weeklyTimes[]` and `weeklyPeriods[]` each carry `description`/`startDate`/`endDate` plus one entry per weekday (`{ open, hours }`): in `weeklyTimes` `hours` is a flat list of time-of-day strings, in `weeklyPeriods` a list of `[start, end]` pairs. Which appears depends on the supplier. |
| `itinerary` | `{ type, text }` | `type` seen as `UNSTRUCTURED`; free-text itinerary. Present or absent per product, the same on all three endpoints. |

## Policy and inclusion fields

| Field | Type | Notes |
| --- | --- | --- |
| `isCancellable` | boolean | |
| `cancellationCutOff` | string | Free-text policy, e.g. "full refund if more than 24h before start". |
| `cancellationCutOffAmount` / `cancellationCutOffUnit` | int32 / string | Structured cut-off. |
| `modalitiesIncluded` / `modalitiesExcluded` / `modalitiesAdditionalInfo` | string | Free-text inclusions and exclusions. |
| `additionalInformation` | object | Free-form key/value map (e.g. `"Meeting Point"`) plus an `unstructured_data` string of newline-joined notices. Keys vary per product. |
| `handiAccess` | boolean | Accessibility flag. |

## Rating and ranking fields

| Field | Type | Notes |
| --- | --- | --- |
| `rating` | double | 0-5. |
| `numberOfRatings` | int32 | |
| `bestseller` | boolean | |
| `relevanceScore` | double | Recommendation score. Absent on get product. |
| `descriptors` | `{ type, reason }[]` | Marketing tags, e.g. `[{ "type": "PROMOTED", "reason": "PROMOTED" }]`. Empty on nearly all products. |

## Classification fields

| Field | Type | Notes |
| --- | --- | --- |
| `categories` | `object[]` | Product-level category codes. Each item follows [Category](#category). |
| `taxonomies` | `object[][]` | 2D: each inner array is one path through the taxonomy tree, ordered root to leaf. Each node follows [Taxonomy node](#taxonomy-node). Iterate the outer array as separate paths; don't flatten. |

### Category

| Field | Type |
| --- | --- |
| `level` | int32 |
| `code` | string |

### Taxonomy node

| Field | Type | Notes |
| --- | --- | --- |
| `uuid` | string | |
| `name` | string | |
| `level` | int32 | |
| `slug` | string | E.g. `activities-culture`. |
| `family` | string | E.g. `activities`. |
| `parentUuid` | string | |
| `nameFullPath` | string | Full breadcrumb, e.g. `"Culture#Tickets & passes"`. |
| `nameTranslation` | string | `name` translated into `displayedLanguage`. |
| `startDate` | string | ISO 8601; when this node took effect. |
| `value` | double | Confidence/weight. |
| `version` | int32 | |
| `isCurrent` | boolean | |

## Supplier and record fields

| Field | Type | Notes |
| --- | --- | --- |
| `supplier` | `{ id, name, externalId, creationDate, updateDate?, contact }` | The supplying partner, not the booking engine. |
| `creationDate` / `updateDate` | string | ISO 8601 with offset. Catalogue record timestamps, not availability dates. |
