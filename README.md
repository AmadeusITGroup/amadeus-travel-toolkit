# amadeus-travel-toolkit

A plugin marketplace bringing Amadeus travel expertise to AI agents.

## Quickstart

In **Claude Code**:

```
# add marketplace
/plugin marketplace add AmadeusITGroup/amadeus-travel-toolkit

# install amadeus-travel-builder plugin
/plugin install amadeus-travel-builder@amadeus-travel-toolkit
```

Using a different agent? See [Installation](#installation).

## Plugins

<table>
  <thead>
    <tr>
      <th width="260" align="left">Plugin</th>
      <th align="left">What it does</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><a href="plugins/amadeus-travel-builder/"><code>amadeus-travel-builder</code></a></td>
      <td>Build applications including travel-related capabilities on live Amadeus data with Amadeus Travel Builder APIs. Expose flight search, tour and activity recommendations, real-time pricing and booking links, airport and city lookup, and other travel capabilities with no API docs to read first.</td>
    </tr>
  </tbody>
</table>

See each plugin's README for its skills and usage.

## Installation

Claude Code is covered in the [Quickstart](#quickstart). Expand your agent below, or [install the skills directly](#skills-all-or-individual) to pick individual skills or target any other agent.

<details>
<summary><b>GitHub Copilot CLI</b></summary>

```
# add marketplace
/plugin marketplace add AmadeusITGroup/amadeus-travel-toolkit

# install amadeus-travel-builder plugin
/plugin install amadeus-travel-builder@amadeus-travel-toolkit
```

</details>

<details>
<summary><b>Cursor CLI</b></summary>

```
/plugin marketplace add https://github.com/AmadeusITGroup/amadeus-travel-toolkit
```

Then select the plugin you want in the plugin picker to install it.

</details>

<details>
<summary><b>Codex CLI</b></summary>

```
/plugins
```

Enter the marketplace URL `https://github.com/AmadeusITGroup/amadeus-travel-toolkit`, then install the plugin you want.

</details>

<details>
<summary><b>Antigravity</b></summary>

```
agy plugin install https://github.com/AmadeusITGroup/amadeus-travel-toolkit
```

</details>

### Skills (all or individual)

Install the skills directly with the [Vercel Skills CLI](https://github.com/vercel-labs/skills) - useful for installing every skill, just one, or targeting any agent:

```bash
# Interactive - pick the skills and agents you want
npx skills add https://github.com/AmadeusITGroup/amadeus-travel-toolkit

# All skills, for a given agent
npx skills add https://github.com/AmadeusITGroup/amadeus-travel-toolkit --skill '*' -a <agent>

# A single skill - see the plugin's README for the skills it ships
npx skills add https://github.com/AmadeusITGroup/amadeus-travel-toolkit --skill <skill-name> -a <agent>
```

`-a` accepts a wide range of agents (e.g. `claude-code`, `cursor`, `codex`, `opencode`, `github-copilot`) - run `npx skills add --help` for the full list.

## Authentication

Applications built with these skills call Amadeus services authenticated with an API key. Don't have one yet? [Request API access](https://eu1.ironcladapp.com/public-launch/6aaa463f6f95614fe0f6db9d).

## Support

If you need help or encounter issues with these skills, search for existing issues or open a new one in the [GitHub Issue Tracker](../../issues).

## Contributing

We welcome your feedback. We don't accept pull requests - the skills are verified against the live APIs by the teams that own them - but you can help by:

- [Reporting bugs or inaccuracies](../../issues) in the skill Markdown files.
- Suggesting new skills to add to this repository by filing a feature request.

See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

## License

Copyright 2026 Amadeus IT Group.

You are free to copy, modify, and distribute these skills under the terms of the
Apache 2.0 license. See the [LICENSE](LICENSE) file for details.
