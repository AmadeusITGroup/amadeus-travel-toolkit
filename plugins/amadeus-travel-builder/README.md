# amadeus-travel-builder

Build applications with **Amadeus Travel Builder APIs** using your AI coding agent. The plugin ships a set of *skills* - markdown playbooks the agent loads on demand to map your prompt onto the right Amadeus endpoints, request shapes, and integration flows.

The Amadeus Travel Builder APIs are free to use, with some limits on content and usage. For example, flight search covers only a subset of airlines.

## Install

In Claude Code:

```
/plugin marketplace add AmadeusITGroup/amadeus-travel-toolkit
/plugin install amadeus-travel-builder@amadeus-travel-toolkit
```

For other agents, see the [marketplace README](../../README.md#installation).

## What's in it

| Skill | What you can build |
| --- | --- |
| [`flight-search`](skills/flight-search/SKILL.md) | Flight comparison and fare-aggregation features - one-way, return, multi-city. Priced offers with merchant deep links. |
| [`activity-search`](skills/activity-search/SKILL.md) | Tours-and-activities feeds - top activities plus smart, filterable recommendations. |
| [`airline-code-lookup`](skills/airline-code-lookup/SKILL.md) | Airline-code lookups - resolve IATA / ICAO codes to human-readable airline names. |
| [`airport-city-search`](skills/airport-city-search/SKILL.md) | Flight-booking location pickers and autocomplete - airport and city search by name, plus by-ID lookup. |
| [`nearest-airports`](skills/nearest-airports/SKILL.md) | Nearest-airport lookups - the most relevant airports near a lat/lon. |
| [`city-search`](skills/city-search/SKILL.md) | City search and autocomplete - by name, including cities with no airport, with optional airport expansion. |

## How to use

Prompt your agent in plain English, describing what you want to build. It matches your prompt against each skill's `description` and loads the matching skill before writing the implementation.

- *"Build me a flight comparison page for round-trips"* → `flight-search`, plus `airline-code-lookup` and `airport-city-search`.
- *"Build a things-to-do feed that recommends top activities"* → `activity-search`.
- *"Build an airport picker that shows the airports within 100 km of the user's GPS coordinates"* → `nearest-airports`.

## Authentication

Applications built with these skills call Amadeus Travel Builder APIs authenticated with an API key. Don't have one yet? [Request API access](https://eu1.ironcladapp.com/public-launch/6aaa463f6f95614fe0f6db9d).

## Feedback

Open an issue in the [GitHub Issue Tracker](https://github.com/AmadeusITGroup/amadeus-travel-toolkit/issues).

## License

Apache 2.0 license. See the [LICENSE](LICENSE) file for details.
