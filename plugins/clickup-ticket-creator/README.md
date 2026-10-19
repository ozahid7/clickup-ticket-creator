# clickup-ticket-creator

Creates ClickUp tickets in the house format via `/ticket-creator`.

## Install

```
/plugin marketplace add ozahid7/ozahid-marketplace
/plugin install clickup-ticket-creator@ozahid-marketplace
```

Then run `/mcp` and authenticate the `clickup` server.

The plugin bundles two MCP servers in `.mcp.json`:

- `clickup`: https://mcp.clickup.com/mcp (OAuth)
- `playwright`: `npx @playwright/mcp@latest` (only used for screenshots, needs Node)

## Already have them?

If you already have a ClickUp connector or the Playwright plugin, disable the bundled duplicates in `/mcp`.

Manual install instead of the bundle:

```
claude mcp add --transport http clickup https://mcp.clickup.com/mcp
/plugin install playwright@claude-plugins-official
```
