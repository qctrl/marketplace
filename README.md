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

## Repository layout

```text
.claude-plugin/
  marketplace.json      # Marketplace catalog. Lists every plugin.
plugins/
  <plugin-name>/        # One folder per plugin.
    .claude-plugin/
      plugin.json       # Plugin manifest.
      icon.png          # Listing icon. Square PNG, 512 to 2048 px.
    README.md           # Plugin docs. Required for directory submission.
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
     "version": "0.1.0",
     "author": {
       "name": "Q-CTRL"
     },
     "license": "Apache-2.0"
   }
   ```

   Use kebab-case for `name` and never change it after release. Raise `version` on every release.

2. Create `plugins/<plugin-name>/README.md`. Use at least 40 words, not counting code blocks. Describe everything the plugin runs, sends, or fetches. Include the install line:

   ```shell
   /plugin install <plugin-name> --marketplace qctrl/marketplace
   ```

3. Add the plugin's skills, agents, hooks, or MCP servers to `plugins/<plugin-name>/`.

4. Add an entry to the `plugins` array in `.claude-plugin/marketplace.json`:

   ```json
   {
     "name": "<plugin-name>",
     "source": "./plugins/<plugin-name>",
     "description": "What the plugin does."
   }
   ```

5. Validate the marketplace and the plugin:

   ```shell
   claude plugin validate .
   claude plugin validate ./plugins/<plugin-name> --profile directory
   ```

## Submitting a plugin to the Claude directory

The [Claude plugin directory](https://claude.com/docs/plugins/submit) accepts each plugin folder as its own submission. Before you submit, make sure the plugin:

- Has a `name` that does not start with `claude-` or `anthropic-`, and is not a generic name such as `plugin`, `mcp`, or `test`.
- Has `description`, `version`, `author`, and `license` in `plugin.json`, or a `LICENSE` file in the plugin folder.
- Has a `README.md` of at least 40 words.
- Has a square PNG icon at `.claude-plugin/icon.png`, 512 to 2048 px on each side and under 2 MB. The directory uses the icon from the first submission only, so add it before you submit.
- Pins exact package versions in `npx` or `uvx` commands, for example `npx <package>@1.2.3`.
- Contains no credentials. Use `userConfig` with `"sensitive": true` instead.
- Keeps every file it runs inside the plugin folder.

Then submit the plugin folder at [claude.ai/directory/manage](https://claude.ai/directory/manage) and select **Validate** to run the full directory checks. See the [pre-submission checklist](https://claude.com/docs/plugins/pre-submission-checklist) for all rules.

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
