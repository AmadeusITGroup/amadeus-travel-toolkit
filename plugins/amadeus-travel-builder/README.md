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

<table>
  <thead>
    <tr>
      <th width="200" align="left">Skill</th>
      <th align="left">What you can build</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><a href="skills/flight-search/SKILL.md"><code>flight-search</code></a></td>
      <td>Flight comparison and fare-aggregation features - one-way, return, multi-city. Priced offers with merchant deep links.</td>
    </tr>
    <tr>
      <td><a href="skills/activity-search/SKILL.md"><code>activity-search</code></a></td>
      <td>Tours-and-activities feeds - top activities plus smart, filterable recommendations.</td>
    </tr>
    <tr>
      <td><a href="skills/airline-code-lookup/SKILL.md"><code>airline-code-lookup</code></a></td>
      <td>Airline-code lookups - resolve IATA / ICAO codes to human-readable airline names.</td>
    </tr>
    <tr>
      <td><a href="skills/airport-city-search/SKILL.md"><code>airport-city-search</code></a></td>
      <td>Flight-booking location pickers and autocomplete - airport and city search by name, plus by-ID lookup.</td>
    </tr>
    <tr>
      <td><a href="skills/nearest-airports/SKILL.md"><code>nearest-airports</code></a></td>
      <td>Nearest-airport lookups - the most relevant airports near a lat/lon.</td>
    </tr>
    <tr>
      <td><a href="skills/city-search/SKILL.md"><code>city-search</code></a></td>
      <td>City search and autocomplete - by name, including cities with no airport, with optional airport expansion.</td>
    </tr>
  </tbody>
</table>

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
