---
name: ticket-creator
description: Create a ClickUp ticket in the house format (assignee, type, title, priority, type-specific description, GitHub branch/PR linking, optional Playwright screenshots). Use whenever the user types /ticket-creator or /ticket, or asks to create, open, log or file a ticket, task, bug or issue in ClickUp, or to make a ticket from a PR, even if they don't say "ClickUp".
---

# Ticket creator

Create one ClickUp ticket from `$ARGUMENTS` (free text). The text may also contain a PR number or URL, a page URL, a screenshot path, a priority, a list name, or the words "screenshots" / "images".

Defaults:
- No default list. Use the list named in the arguments; if none is named, call `clickup_get_workspace_hierarchy` and ask the user which list (step 0).
- Needs only `git` and the ClickUp MCP. Never use `gh` or any CLI that has to be installed first.
- Load the ClickUp tools with ToolSearch before first use (`clickup_get_workspace_hierarchy`, `clickup_get_list`, `clickup_create_task`, `clickup_update_task`, `clickup_find_member_by_name`, `clickup_resolve_assignees`, `clickup_request_attachment_upload`). Load Playwright tools only when screenshots are requested.

## Workflow

Follow in order. Stop and ask the user only where a step says so.

### 0. Ask first (always)
Before any other step, always ask the user two questions, even when the arguments already mention a column or a type. Do not create a ticket, branch or PR edit before both are answered.
0. Resolve the target list. If the arguments name one, find it with `clickup_get_workspace_hierarchy`. Otherwise call `clickup_get_workspace_hierarchy` and ask the user which list with `AskUserQuestion` (max 4 options; the rest stay reachable via "Other").
1. Call `clickup_get_list` on the target list and read `statuses` in `orderindex` order. These are the board columns, fetched live so new columns appear automatically.
2. Make one `AskUserQuestion` call with two questions:
   - **Column**: which column the ticket goes in. One option per status (max 4; if there are more, show the first 4 and the rest stay reachable via "Other").
   - **Type**: Bug, Hotfix, Feature, Enhancement, Refactor, Chore or Docs. The tool shows at most 4 options, so offer the 4 most fitting types for the request, best guess first marked "(Recommended)", and name all 7 types in the question text so the others can be typed via "Other".
3. The answers are final: the column is the ticket status, the type drives step 2.

### 1. Assignee
Always assign to the user, never to anyone else.
1. Run `git config user.name` and `git config user.email`.
2. Match a ClickUp workspace member by email first, then by name (`clickup_find_member_by_name`, `clickup_resolve_assignees`).
3. No match: stop and ask. Do not create the ticket unassigned.

### 2. Type
Use the type answered in step 0. Exactly one:

| Type        | When                                             | Branch prefix |
|-------------|--------------------------------------------------|---------------|
| Bug         | Something is broken or behaves incorrectly       | bug/          |
| Hotfix      | Critical production bug that needs an urgent fix | hotfix/       |
| Feature     | New functionality that didn't exist before       | feature/      |
| Enhancement | Improvement to existing functionality            | enhance/      |
| Refactor    | Code change with no behavior change              | refactor/     |
| Chore       | Tooling, dependencies, config, CI                | chore/        |
| Docs        | Documentation only                               | docs/         |

The type is set as a lowercase tag on the ticket (`bug`, `hotfix`, `feature`, `enhancement`, `refactor`, `chore`, `docs`).

### 3. Title
`[Type] Area: short specific description`, about 80 characters at most. Say what and where. Example: `[Bug] Settings: Save button does nothing when email is unchanged`. No vague titles like "fix bug".

### 4. Priority
Use what the user said. Otherwise infer:
- Urgent: production down, data loss, security, or type Hotfix
- High: blocks users or other work
- Low: cosmetic or nice-to-have
- Normal: everything else (default)

### 5. Source: normal flow or existing PR
- **Case A, ticket first:** the description comes from the user's text.
- **Case B, PR already exists:** the arguments name a PR (number or URL), or the user says the ticket is for the current PR. Read it with git only, and use that content to fill the description:
  - Current branch: branch from `git branch --show-current`, content from `git log origin/main..HEAD` and `git diff origin/main...HEAD` (use the repo's default branch if not `main`).
  - PR number given: `git fetch origin pull/<n>/head`, then `git log` and `git diff` of `origin/main...FETCH_HEAD`. Branch name: the entry in `git ls-remote --heads origin` with the same sha as FETCH_HEAD, else "Unknown".
  - PR link: the URL the user gave, else `https://github.com/<owner>/<repo>/pull/<n>` built from `git remote get-url origin`. If only "current PR" was said and the number is unknown, ask the user for it.

### 6. Description
Use the template for the type. Facts only: write "Unknown" for anything missing, never guess. Write it so a stranger understands it six months from now.

**Bug / Hotfix**
```
**Description**
What is wrong and its impact.

**Steps to reproduce**
1.
2.
3.

**Expected result**

**Actual result**
(error messages verbatim)

**Environment**
- Env: local / staging / prod
- Branch/commit:
- Browser/OS:
- User role:

**Evidence**
Screenshots (only if requested), console errors, logs.

**Frequency:** always / sometimes / once
```

**Feature / Enhancement**
```
**Context**
Why this is needed: the problem or user need.

**Scope**
What to build or change.

**Acceptance criteria**
- [ ]
- [ ]

**Out of scope**
```

**Refactor / Chore / Docs**
```
**What**

**Why**

**Acceptance criteria**
- [ ]
```

Every ticket ends with:
```
**GitHub**
- Branch:
- PR: #<number> <link>
```
Fill what exists, leave the rest as "None yet".

### 7. Create
Call `clickup_create_task` with the list, title, `markdown_description`, priority, the assignee id, `status: <column from step 0>` and `tags: [<type>]`.

If the call fails because the tag does not exist in the space, retry once without tags and tell the user: "Tag `<type>` is missing in the space, add it in ClickUp space settings." Do not stop the flow.

### 8. GitHub linking (the ticket references the PR, the PR references the ticket)
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
Never take or attach screenshots on your own. Skip this step unless the user asked for images or screenshots.

- **User gave a file path:** attach that file and skip Playwright.
- **Otherwise take them with the Playwright MCP:**
  1. Open the relevant page (the URL given, or the page related to the ticket).
  2. Reproduce the steps if needed to reach the state to capture.
  3. Screenshot the relevant element or area, not the whole page, unless a full page was requested.
  4. Save to `./screenshots/CU-<id>-<short-slug>.png`, numbered `-1`, `-2` when there are several.
- **Attach as files, never inside the description.** For each file: call `clickup_request_attachment_upload` with the task id and file name, then POST it with curl, using an argument list and the ticket from the result:
  `curl -s -X POST <upload_url> -H "X-Upload-Ticket: <ticket>" -F "attachment=@<path>;type=image/png"`
  The ticket expires in a few minutes, so request it right before each upload.
- **Evidence section:** list each attached screenshot with one line saying what it shows, then update the ticket description with it.
- **Failures:** if Playwright is not connected or the page can't be reached, say so and keep the ticket without screenshots. If the upload returns `Over allocated storage` (`GBUSED_005`), the workspace storage quota is full, not the file too big: report it, keep the files on disk and give the path.

### 10. Reply
Reply with only:
- Title
- Type
- Column
- Priority
- Ticket link, as an inline markdown link `[title](url)`, never a bare URL
- Branch / PR, if created

Add one extra line only for a warning the user must act on (missing tag, upload failure).

## Rules
- One ticket = one problem. If the request holds several, ask which one first or create separate tickets only after the user agrees.
- Facts only. Don't guess; "Unknown" is a valid value.
- Don't add screenshots, branches, PR edits or any other extra the user didn't ask for, except where steps 8 and 9 say so.
