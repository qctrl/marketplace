# Q-CTRL Claude Code Marketplace

A [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) for Q-CTRL plugins.

## Installation

Add the marketplace in Claude Code:

```shell
/plugin marketplace add qctrl/marketplace
```

Then install a plugin from it:

```shell
/plugin install <plugin-name>@qctrl
```

## Plugins

| Plugin | What it does |
| --- | --- |
| [`fire-opal`](./plugins/fire-opal) | Run Q-CTRL Fire Opal through the Q-CTRL MCP server: validate and run circuits, estimate expectation values, and run QAOA, dynamics and Monte Carlo workloads. |

## Repository layout

```text
.claude-plugin/
  marketplace.json      # Marketplace catalog. Lists every plugin.
plugins/
  <plugin-name>/        # One folder per plugin.
    .claude-plugin/
      plugin.json       # Plugin manifest.
    skills/             # Optional: skills (skills/<name>/SKILL.md).
    agents/             # Optional: subagents.
    hooks/              # Optional: hooks (hooks/hooks.json).
    .mcp.json           # Optional: MCP servers.
```

## Adding a plugin

1. Create `plugins/<plugin-name>/.claude-plugin/plugin.json`:

   ```json
   {
     "name": "<plugin-name>",
     "description": "What the plugin does.",
     "version": "0.1.0"
   }
   ```

2. Add the plugin's skills, agents, hooks, or MCP servers to `plugins/<plugin-name>/`.

3. Add an entry to the `plugins` array in `.claude-plugin/marketplace.json`:

   ```json
   {
     "name": "<plugin-name>",
     "source": "./plugins/<plugin-name>",
     "description": "What the plugin does."
   }
   ```

4. Validate:

   ```shell
   claude plugin validate .
   ```

## Testing locally

Add your local checkout as a marketplace:

```shell
/plugin marketplace add ./path/to/marketplace
```

## Contributing

See [Contributing](https://code.q-ctrl.com/contributing).

## Credits

See [Contributors](https://github.com/qctrl/marketplace/graphs/contributors).

## Version

This project follows the [Semantic Versioning](https://semver.org/) standard.
