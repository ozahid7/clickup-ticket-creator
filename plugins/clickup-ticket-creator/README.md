# clickup-ticket-creator

Creates ClickUp tickets in the house format via `/ticket-creator`.

## Install

```
/plugin marketplace add ozahid7/clickup-ticket-creator
/plugin install clickup-ticket-creator@ozahid-marketplace
```

Run it with `/clickup-ticket-creator:ticket-creator` or just ask Claude to create a ticket.

Then run `/mcp` and authenticate the `clickup` server.

The plugin bundles two MCP servers in `.mcp.json`:

- `clickup`: https://mcp.clickup.com/mcp (OAuth)
- `playwright`: `npx @playwright/mcp@latest` (only used for screenshots, needs Node)

The first run saves your list and assignee to `.claude/ticket-creator.local.json` (excluded from git) so later runs skip the workspace lookup. Delete that file to reset.

## Install in Codex

```
codex plugin marketplace add ozahid7/clickup-ticket-creator
codex plugin add clickup-ticket-creator@ozahid-marketplace
```

Authenticate the `clickup` MCP server on first use, then invoke it with `$ticket-creator` or just ask Codex to create a ticket.

## Already have them?

If you already have a ClickUp connector or the Playwright plugin, disable the bundled duplicates in `/mcp`.

Manual install instead of the bundle:

```
claude mcp add --transport http clickup https://mcp.clickup.com/mcp
/plugin install playwright@claude-plugins-official
```
