# clickup-ticket-creator

Creates ClickUp tickets in the house format via `/ticket-creator`. Works for code tickets (bug, feature, enhancement, refactor, chore, docs, infra) and non-code tickets (design, research, meeting, task). Git is optional: branch and PR linking only apply to code tickets inside a git repo.

## What you need

- A ClickUp account with access to the list where tickets go.
- Node.js, only if you want screenshots (the Playwright server runs through `npx`).
- Git, only for branch and PR linking on code tickets.

The plugin bundles two MCP servers in `.mcp.json`:

- `clickup`: https://mcp.clickup.com/mcp (OAuth)
- `playwright`: `npx @playwright/mcp@latest` (only used for screenshots)

## Claude Code

1. Install the Claude Code CLI:

   ```
   curl -fsSL https://claude.ai/install.sh | bash
   ```

   Or with npm: `npm install -g @anthropic-ai/claude-code`. Check it with `claude --version`, then run `claude` once to log in.

2. Add the marketplace:

   ```
   claude plugin marketplace add ozahid7/clickup-ticket-creator
   ```

3. Install the plugin:

   ```
   claude plugin install clickup-ticket-creator@ozahid-marketplace
   ```

4. Check it worked:

   ```
   claude plugin list
   claude mcp list
   ```

5. Authenticate ClickUp: start `claude`, run `/mcp`, select the `clickup` server, choose Authenticate and finish the OAuth login in your browser. This step only works inside a session.
6. Create a ticket, straight from the shell:

   ```
   claude "/clickup-ticket-creator:ticket-creator login button is broken on Safari"
   ```

   Or start `claude` and ask: "create a ticket for ...".

Update and remove:

```
claude plugin marketplace update ozahid-marketplace
claude plugin update clickup-ticket-creator@ozahid-marketplace
claude plugin uninstall clickup-ticket-creator@ozahid-marketplace
```

## Codex

1. Add the marketplace:

   ```
   codex plugin marketplace add ozahid7/clickup-ticket-creator
   ```

2. Install the plugin, either from the terminal:

   ```
   codex plugin add clickup-ticket-creator@ozahid-marketplace
   ```

   or inside Codex: run `/plugins`, open ClickUp Ticket Creator and select Install plugin.

3. Authenticate the `clickup` MCP server when Codex asks (at install or on first use) and finish the OAuth login in your browser.
4. Create a ticket:

   ```
   $ticket-creator login button is broken on Safari
   ```

   You can also just ask: "create a ticket for ...". Codex asks its questions in plain chat.

Update: `codex plugin marketplace upgrade`. If a command is rejected, check `codex plugin --help` for the syntax of your version.

## Cursor

Pick one install path.

Team marketplace (Cursor Team or Enterprise):

1. An admin opens Dashboard -> Plugins & MCPs -> Add Marketplace -> Import from Repo and enters `ozahid7/clickup-ticket-creator`.
2. Each user opens Customize in the sidebar, finds ClickUp Ticket Creator and selects Install.

Local install (any plan):

1. Clone this repo.
2. Copy the plugin folder:

   ```
   mkdir -p ~/.cursor/plugins/local
   cp -r plugins/clickup-ticket-creator ~/.cursor/plugins/local/clickup-ticket-creator
   ```

3. Run Developer: Reload Window from the command palette.
4. Open Customize and check the plugin is listed.

Then, for both paths:

1. Open Cursor Settings -> MCP, find `clickup` and log in. Finish the OAuth login in your browser.
2. In Agent chat ask: "create a ticket for login button is broken on Safari".

## First run

The same flow runs in every tool:

1. If no list is saved yet, it asks which ClickUp list to use (or name it in your request).
2. It asks which column the ticket goes in and what type it is (Bug, Hotfix, Feature, Enhancement, Refactor, Chore, Docs, Design, Infra, Research, Meeting, Task).
3. It fills the matching template and creates the ticket.
4. For code tickets in a git repo it can link the branch or PR (mention a PR number or URL).
5. Add the word "screenshots" to your request to attach Playwright screenshots.

Your list and assignee are saved to `.claude/ticket-creator.local.json` (excluded from git) in the repo, or `~/.claude/ticket-creator.local.json` outside a repo, so later runs skip the lookup. Delete that file to reset.

## Troubleshooting

- ClickUp tools missing or unauthorized: re-authenticate the `clickup` server in `/mcp` (Claude Code), the Codex prompt, or Cursor Settings -> MCP.
- Screenshots fail: install Node.js so `npx` is available.
- Tickets land in the wrong list: name the list in your request or delete the cache file.

## Already have them?

If you already have a ClickUp connector or the Playwright plugin, disable the bundled duplicates in `/mcp`.

Manual install instead of the bundle:

```
claude mcp add --transport http clickup https://mcp.clickup.com/mcp
claude plugin install playwright@claude-plugins-official
```
