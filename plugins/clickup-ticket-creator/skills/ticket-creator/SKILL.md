---
name: ticket-creator
description: Create a ClickUp ticket in the house format (assignee, type, title, priority, type-specific description, optional GitHub branch/PR linking, optional Playwright screenshots). Works for code tickets (bug, feature, refactor, infra...) and non-code tickets (design, research, meetings, general tasks) with or without a git repo. Use whenever the user types /ticket-creator or /ticket, or asks to create, open, log or file a ticket, task, bug or issue in ClickUp, or to make a ticket from a PR, even if they don't say "ClickUp".
allowed-tools: mcp__plugin_clickup-ticket-creator_clickup, Bash(git log:*), Bash(git diff:*), Bash(git branch:*), Bash(git remote:*), Bash(git fetch:*), Bash(git ls-remote:*), Bash(git config:*), Bash(git symbolic-ref:*), Bash(git rev-parse:*)
---

# Ticket creator

Create one ClickUp ticket from `$ARGUMENTS` (free text). The text may also contain a PR number or URL, a page URL, a screenshot path, a priority, a list name, or the words "screenshots" / "images".

Defaults:
- Needs only the ClickUp MCP. `git` is used only for code tickets inside a git repo (check with `git rev-parse --is-inside-work-tree`). Never use `gh` or any CLI that has to be installed first.
- If AskUserQuestion or ToolSearch is not available (e.g. Codex), ask the same questions in plain chat and call the ClickUp tools directly; treat the user's request as $ARGUMENTS.
- Load the ClickUp tools with ONE ToolSearch call (`select:clickup_get_list,clickup_create_task,clickup_update_task,clickup_resolve_assignees,clickup_find_member_by_name,clickup_get_workspace_hierarchy,clickup_request_attachment_upload`). Load Playwright tools only when screenshots are requested.
- Run independent calls in the same turn (list fetch, assignee lookup, git commands). Only sequence calls that depend on each other.

## Cache

File: `.claude/ticket-creator.local.json` in the repo root when inside a git repo, else `~/.claude/ticket-creator.local.json`.

```json
{ "listId": "", "listName": "", "assigneeId": "" }
```

- Read it first. When present, it replaces the hierarchy fetch and the assignee lookup.
- Write it after the first successful run. In a repo, also append `.claude/ticket-creator.local.json` to `.git/info/exclude` so it is never committed.
- Ignore the cached list when the arguments name a different list, and overwrite the file with the new one.
- If a ClickUp call fails because the list or member id is unknown, delete the cache and redo the lookup once.

## Workflow

Follow in order. Stop and ask the user only where a step says so.

### 0. Ask first (always)
Before any other step, always ask the user two questions, even when the arguments already mention a column or a type. Do not create a ticket, branch or PR edit before both are answered.
0. Resolve the target list. Use the list named in the arguments, else the cached list. If neither exists, call `clickup_get_workspace_hierarchy` and ask the user which list with `AskUserQuestion` (max 4 options; the rest stay reachable via "Other").
1. Call `clickup_get_list` on the target list and read `statuses` in `orderindex` order. These are the board columns, fetched live so new columns appear automatically. Fetch the assignee (step 1) in the same turn.
2. Make one `AskUserQuestion` call with two questions:
   - **Column**: which column the ticket goes in. One option per status (max 4; if there are more, show the first 4 and the rest stay reachable via "Other").
   - **Type**: Bug, Hotfix, Feature, Enhancement, Refactor, Chore, Docs, Design, Infra, Research, Meeting or Task. The tool shows at most 4 options: first the best guess for the request marked "(Recommended)", then fill the rest from Bug, Enhancement, Feature, Refactor, Design in that order, skipping the guess. Name all 12 types in the question text so the others can be typed via "Other".
3. The answers are final: the column is the ticket status, the type drives step 2.

### 1. Assignee
Always assign to the user, never to anyone else.
1. Use `assigneeId` from the cache when present.
2. Otherwise call `clickup_resolve_assignees` with `"me"`. If that returns no match and you are in a git repo, run `git config user.name` and `git config user.email` and match a workspace member by email first, then by name (`clickup_find_member_by_name`). Outside a repo, ask the user for their ClickUp name.
3. No match: stop and ask. Do not create the ticket unassigned.

### 2. Type
Use the type answered in step 0. Exactly one:

| Type        | When                                              | Branch prefix | Template                   |
|-------------|---------------------------------------------------|---------------|----------------------------|
| Bug         | Something is broken or behaves incorrectly        | bug/          | `templates/bug.md`         |
| Hotfix      | Critical production bug that needs an urgent fix  | hotfix/       | `templates/bug.md`         |
| Feature     | New functionality that didn't exist before        | feature/      | `templates/feature.md`     |
| Enhancement | Improvement to existing functionality             | enhance/      | `templates/feature.md`     |
| Refactor    | Code change with no behavior change               | refactor/     | `templates/maintenance.md` |
| Chore       | Tooling, dependencies, config, CI                 | chore/        | `templates/maintenance.md` |
| Docs        | Documentation only                                | docs/         | `templates/maintenance.md` |
| Infra       | Cloud, deploy, servers, monitoring, IaC, access   | infra/        | `templates/infra.md`       |
| Design      | UI/UX, mockups, Figma, visual work                | none          | `templates/design.md`      |
| Research    | Investigation, spike, comparison, decision needed | none          | `templates/task.md`        |
| Meeting     | Meeting, workshop, demo prep, follow-ups          | none          | `templates/task.md`        |
| Task        | Anything else: admin, planning, ops, process      | none          | `templates/task.md`        |

The type is set as a lowercase tag on the ticket (`bug`, `hotfix`, `feature`, `enhancement`, `refactor`, `chore`, `docs`, `infra`, `design`, `research`, `meeting`, `task`).

**Code tickets** are Bug, Hotfix, Feature, Enhancement, Refactor, Chore, Docs and Infra. **Non-code tickets** are Design, Research, Meeting and Task. Steps 5 (Case B) and 8 apply only to code tickets inside a git repo. For all others, skip them: no Affected code, Changes, GitHub section or branch name. For a code ticket outside a git repo, also skip them and add the reply line "Not a git repo, GitHub linking skipped".

### 3. Title
`[Type] Area: short specific description`, about 80 characters at most. Say what and where. Example: `[Bug] Settings: Save button does nothing when email is unchanged`. No vague titles like "fix bug". For non-code tickets, Area is the team, project or topic: `[Meeting] Q4 Planning: prepare roadmap review`.

### 4. Priority
Use what the user said. Otherwise infer:
- Urgent: production down, data loss, security exposure, infra outage, or type Hotfix
- High: blocks users or other work
- Low: cosmetic or nice-to-have
- Normal: everything else (default)

### 5. Source: normal flow or existing PR
- **Case A, ticket first:** the description comes from the user's text. Always the case for non-code tickets and outside a git repo.
- **Case B, PR already exists** (code tickets in a git repo only): the arguments name a PR (number or URL), or the user says the ticket is for the current PR. Read it with git only, and use that content to fill the description. Run the git commands in parallel.
  - Default branch: `git symbolic-ref --short refs/remotes/origin/HEAD` (e.g. `origin/main`). Call it `<base>` below.
  - Read cheap first: `git log --format='%s%n%b' <base>..HEAD` and `git diff --stat <base>...HEAD`. Open the full diff of specific files only when that is not enough to describe the change.
  - The `--stat` file list feeds **Affected code**. The log feeds **Changes**: 3 to 6 bullets summarizing what the commits did.
  - Current branch: branch from `git branch --show-current`, content from the commands above.
  - PR number given: `git fetch origin pull/<n>/head`, then the same commands against `<base>...FETCH_HEAD`. Branch name: the entry in `git ls-remote --heads origin` with the same sha as FETCH_HEAD, else "Unknown".
  - PR link: the URL the user gave, else `https://github.com/<owner>/<repo>/pull/<n>` built from `git remote get-url origin`. If only "current PR" was said and the number is unknown, ask the user for it.

### 6. Description
Read the template file for the type (table in step 2) and fill it. Facts only, never guess. Write it so a stranger understands it six months from now.

Required sections are always shown; write "Unknown" if there is no fact for them:
- Bug / Hotfix: Description, Steps to reproduce, Expected result, Actual result
- Feature / Enhancement: Context, Scope, Acceptance criteria
- Refactor / Chore / Docs / Infra: What, Why
- Design: Context, Deliverables
- Research / Meeting / Task: Description, Goal / expected outcome

Every other section and field (Environment lines, Evidence, Frequency, Out of scope, Affected code, Links, Due date) is omitted when there is no fact for it. Do not write "Unknown" in them.

**Affected code** lists the files, routes or components when the user named them or Case B gives them. **Changes** is Case B only; drop it in Case A.

Code templates end with a **GitHub** section. Fill what exists, leave the rest as "None yet". Design and task templates have none.

### 7. Create
Call `clickup_create_task` with the list, title, `markdown_description`, priority, the assignee id, `status: <column from step 0>` and `tags: [<type>]`.

If the call fails because the tag does not exist in the space, retry once without tags and tell the user: "Tag `<type>` is missing in the space, add it in ClickUp space settings." Do not stop the flow.

After the first successful create, write the cache file (see Cache).

### 8. GitHub linking (code tickets in a git repo only)
The ticket references the PR, the PR references the ticket.

Let `<id>` be the returned task id and `<link>` its URL.

**Case A, ticket first**
- Branch name: `<prefix>CU-<id>-<short-slug>`, for example `bug/CU-86abc123-save-button-settings`.
- Creating the branch changes the user's working tree, so create it (`git switch -c`) only when the user asked for one. Otherwise print the branch name and PR rules and let the user decide. Never open a PR yourself: print the PR title and body for the user to paste.
- PR title: `[CU-<id>] <short description>`.
- PR body: contains `Closes CU-<id>` and `<link>`.
- After a branch or PR exists, update the ticket's GitHub section (`clickup_update_task` with the new `markdown_description`).

**Case B, PR exists**
- Add the PR number and link and the branch to the ticket's GitHub section at creation time.
- Do not edit the PR yourself. Print the exact new PR title `[CU-<id>] <old title>` and the lines to append to the body (`Closes CU-<id>` and `<link>`) so the user pastes them in GitHub.

### 9. Screenshots (only when explicitly asked)
Skip this step unless the user asked for images or screenshots. When they did, read `screenshots.md` in this skill folder and follow it.

### 10. Reply
Reply with only:
- Title
- Type
- Column
- Priority
- Ticket link, as an inline markdown link `[title](url)`, never a bare URL
- Branch / PR, if created (code tickets only)

Add one extra line only for a warning the user must act on (missing tag, upload failure).

## Rules
- One ticket = one problem. If the request holds several, ask which one first. After the user agrees to create several, use `clickup_get_operators` + `clickup_execute_operator` for a bulk create when an operation covers it, instead of N `clickup_create_task` calls.
- Facts only. Don't guess; "Unknown" is a valid value.
- Don't add screenshots, branches, PR edits or any other extra the user didn't ask for, except where steps 8 and 9 say so.
