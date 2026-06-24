---
name: monday-project-setup
description: Sets up the FlowWest Monday.com + GitHub integration for a project. Use this skill whenever someone wants to create a Monday.com project board, connect a GitHub repo to Monday.com, configure the GitHub Actions sync workflows, link GitHub issues to Monday items, or run a backfill to sync current issue state. Trigger even if they only mention one part of the setup (e.g. "I need to set up a Monday board" or "how do I wire up the GitHub sync" or "link my issues to Monday"). This skill handles new projects from scratch, partially set-up projects, and adding GitHub wiring to an existing board.
---

# Monday Project Setup

This skill guides the user through setting up the FlowWest Monday.com + GitHub workflow. The four main phases are: creating or locating a Monday board, wiring the GitHub Actions sync workflows, linking GitHub issues to Monday items, and running a backfill to sync current state. Not every project needs all four — ask the user what they need and skip phases that are already done.

The golden rule throughout: show proposed changes and get explicit approval before writing anything to Monday.

---

## Phase 0: Prerequisites check

Before anything else, verify that the required tools are available. Surface gaps clearly — don't let the user get partway through setup and hit a missing dependency.

**Monday.com MCP** — the skill can't do anything without it. Call `get_user_context` to test the connection. If the tool is absent or the call fails, search the connector registry with `mcp__mcp-registry__search_mcp_registry` and surface the install prompt with `mcp__mcp-registry__suggest_connectors`. Wait for the user to connect before continuing.

**PDF and DOCX skills** — needed only if the user will upload a scope document in Phase 3. Check whether `pdf` (or `anthropic-skills:pdf`) and `docx` (or `anthropic-skills:docx`) appear in the available skills list. If either is missing and the user has files to upload, direct them to **Settings → Capabilities → Skills** to install it. If they have no file to upload, note the gap and move on.

**GitHub MCP** — search the connector registry with `mcp__mcp-registry__search_mcp_registry` using keywords `["github", "issues"]`. If a GitHub connector is available and not yet connected, suggest it — it will allow Claude to fetch issues from private repos directly. If no GitHub MCP exists or the user skips it, note that issue fetching in Phase 5 will be limited to public repos (via web fetch); private repo users will need to provide their issue list manually.

After checking, give the user a one-line status per tool (✅ connected / ⚠️ missing) before moving to Phase 1.

---

## Phase 1: Discover

Ask all of the following in a single message to avoid unnecessary back-and-forth:

- **Project name** — what should the board be called? The naming convention is a short descriptive name followed by the project number in parentheses, e.g. `Spring Run JPE (028-03)`. Ask for both the name and the project number if the user doesn't provide them.
- **Existing Monday board?** — if yes, get the board URL or ID
- **Existing GitHub repo(s)?** — if yes, get the full repo URL(s) (e.g. `https://github.com/FlowWest/my-repo`)
- **Setup scope** — which phases are needed today? Full setup, just GitHub wiring, just issue linking, or just backfill?

Use the answers to decide which phases below apply. It's common for someone to start here and return later to finish remaining phases — that's fine.

---

## Phase 2: Board setup

Skip if the user already has a board — record the existing board ID and move on.

### 2a. Choose creation method

Read `references/board-schema.md` now — you'll need the template board ID, workspace ID, column structure, and required column names to describe the options accurately and to execute whichever the user picks.

Offer two options. The template duplicate is almost always the right choice because it carries over all the column structure the GitHub sync depends on, but some users prefer to start minimal.

> **Option A — Create from template**: Creates a new board in the Data workspace from the FlowWest template center template, including all columns, status labels, story points, and the Issue Progress rollup. Recommended.
>
> **Option B — Blank board with required columns**: Creates an empty board and adds only the three columns the GitHub sync requires: GitHub Issue (link), Status, and Owner.

### 2b. Create the board

**Option A:** Run the `create_board` mutation via `all_monday_api` using `template_id: 22748268` and `workspace_id: 1366388` (Data workspace). Use the full formatted name from Phase 1 (e.g. `Spring Run JPE (028-03)`) as the `board_name`. See `references/board-schema.md` for the exact mutation.

**Option B:** Use `create_board` in workspace `1366388` (Data workspace) with the full formatted name (e.g. `Spring Run JPE (028-03)`). Then add a Subitems column first — Monday won't create the linked subitem board until you do. Once the subitem board exists, use `create_column` to add GitHub Issue (link), Status, and Owner columns on both the main board and the subitem board.

After creation, call `get_board_info` on the new board to capture the actual column IDs and subitem board ID — these will differ from the template values and you'll need them in Phases 4 and 5.

Share the new board URL with the user.

---

## Phase 3: Populate from scope document

Skip if the user only needs GitHub wiring or issue linking.

### 3a. Gather project context

Ask the user for:
1. A scope document (task order, contract, or SOW) as a PDF or Word attachment — use the `pdf` or `docx` skill to read it
2. A brain dump — anything else relevant typed directly in chat (goals, team, milestones, risks)

The scope doc is the authority on deliverables and timeline. The brain dump fills in what the contract doesn't say.

### 3b. Propose structure

Parse the document and propose groups, items, and subitems. The goal is a structure the team will actually use, not an exhaustive decomposition of every line in the contract.

- **Groups**: major phases, work streams, or deliverables — aim for 3–8. Common ones: work stream groups, a Milestones group for key dates, a Communication & planning group for recurring overhead.
- **Items**: things that can be assigned to a person and tracked to completion. Map to specific outputs named in the scope.
- **Subitems**: only where an item has clear sequential sub-tasks worth tracking separately. Don't add subitems just to have them.

Present the structure as an outline:

```
GROUP: Data ingestion
  ├── Item: Connect flow data pipeline     [5 pts]
  │     └── Subitem: Write ingestion script
  │     └── Subitem: QA against historical data
  ├── Item: Validate gauge station coverage [3 pts]

GROUP: Milestones
  ├── Item: Draft report delivered          [End: 2026-09-01]
  ├── Item: Final report delivered          [End: 2026-11-01]
```

Leave Owner blank — the user can assign after creation.

### 3c. Iterate and write

Incorporate any edits the user wants and show the updated structure. Once they approve, write to Monday:
1. `create_group` for each group
2. `create_item` for each item in the correct group
3. `create_item` on the subitem board for items that have subitems
4. `change_item_column_values` to set any dates or story points from the approved structure

Share the board URL when done.

---

## Phase 4: GitHub Actions setup

Skip if the user says GitHub Actions are already configured.

The sync workflows live in `FlowWest/infra` as reusable workflows. Each project repo needs lightweight caller files that reference them, plus the variables and secret that tell the workflows which board to update.

Before diving in, briefly explain what the sync actually does — users who understand the mechanics set things up more carefully:

| GitHub event | Monday update |
|---|---|
| Issue closed | Status → Done |
| Issue reopened | Status → In Progress |
| Label `blocked` added | Status → Stuck |
| Label `blocked` removed | Status → In Progress |
| Issue assigned | Owner set to assignee |
| Issue unassigned | Owner cleared |

The `blocked` label behavior is easy to miss: the user needs to create a label named exactly `blocked` in their GitHub repo for Stuck status to work. Mention this explicitly.

### 4a. Copy the caller files

Ask the user to copy both files from `https://github.com/FlowWest/infra/tree/main/templates` into their project repo at `.github/workflows/`:

```
templates/sync-issue-to-monday.yml   →   .github/workflows/sync-issue-to-monday.yml
templates/backfill-monday-sync.yml   →   .github/workflows/backfill-monday-sync.yml
```

No edits to either file are needed.

### 4b. Generate the .env file

Using the column IDs from Phase 2 (call `get_board_info` on an existing board if you don't have them yet), generate a ready-to-use `monday.env` file and save it to the user's working directory. Both workflows share the same variable values — set them once and both work.

If `get_board_info` isn't enough to surface a particular ID (e.g. for a manually created board), tell the user how to find them in the UI: hover over the column header → click ⋯ → Settings — the column ID appears at the bottom of the panel. For the subitem board ID: click into any subitem row to expand it, then read the second board ID from the URL.

```
MONDAY_BOARD_ID=<main board ID>
MONDAY_ITEM_LINK_COL=<GitHub Issue column ID on items>
MONDAY_ITEM_STATUS_COL=<Status column ID on items>
MONDAY_ITEM_OWNER_COL=<Owner column ID on items>
MONDAY_SUBITEM_BOARD_ID=<subitem board ID>
MONDAY_SUBITEM_LINK_COL=<GitHub Issue column ID on subitems>
MONDAY_SUBITEM_STATUS_COL=<Status column ID on subitems>
MONDAY_SUBITEM_OWNER_COL=<Owner column ID on subitems>
```

See `references/board-schema.md` for the variable names and what each maps to.

### 4c. Set variables and secret

Ask the user whether they'd prefer the `gh` CLI or the GitHub web UI.

**`gh` CLI (faster):**

Check that `gh` is installed (`gh --version`). If not, offer install instructions:
- macOS: `brew install gh`
- Windows: `winget install --id GitHub.cli`
- Linux: https://github.com/cli/cli/blob/trunk/docs/install_linux.md

Authenticate if needed: `gh auth login`

Then for each repo:
```bash
gh variable set --env-file monday.env --repo FlowWest/your-repo
gh secret set MONDAY_API_TOKEN --repo FlowWest/your-repo
```

The `.env` file format is one `KEY=value` per line with no quotes. `gh secret set` reads the token value from stdin so it never appears in shell history — the user will be prompted to paste it after running the command.

`MONDAY_API_TOKEN` is set as a secret (not a variable) because variables are visible to all repo members — tokens must stay private.

**GitHub web UI:**

For each repo: Settings → Secrets and variables → Actions.
- Variables tab → "New repository variable" → add each of the 8 variables from `monday.env`
- Secrets tab → "New repository secret" → add `MONDAY_API_TOKEN`

The Monday API token is the personal API token for the FlowWest Monday bot user. If the user doesn't have it, they should ask the workspace admin.

### 4d. Confirm

Once the files are committed and variables/secret are set, ask the user to go to the repo's Actions tab and confirm both workflow files appear. If they don't, the files are likely not yet on the default branch.

---

## Phase 5: Link GitHub issues to Monday items

Skip if there are no existing GitHub issues to link.

The sync workflow matches issues to Monday items by looking for the issue URL in the GitHub Issue link column. This phase writes those URLs via MCP so the user doesn't have to paste them manually.

### 5a. Gather issues

Ask the user: "Would you like me to pull all open issues from the repo, or do you want to specify particular issue numbers or URLs?"

**If a GitHub MCP is connected**, use it to list issues directly — this works for both public and private repos.

**If no GitHub MCP is available** (GitHub has an official MCP server but it is not currently in the Cowork connector registry, so it won't appear in a registry search): for public repos, fetch issues via `mcp__workspace__web_fetch` at `https://api.github.com/repos/{owner}/{repo}/issues?state=open&per_page=100`. For private repos, this won't work unauthenticated — ask the user to run the following in their terminal and paste the output (they'll already have `gh` set up from Phase 4):

```bash
gh issue list --repo owner/repo --state open --json number,title,url --limit 100
```

If they want specific issues rather than all open ones, accept full URLs or issue numbers (e.g. `#12, #15`) and construct the URLs.

### 5b. Propose mapping

Show a proposed mapping between issues and Monday items:

```
github.com/FlowWest/my-repo/issues/12  →  [Data ingestion] Connect flow data pipeline
github.com/FlowWest/my-repo/issues/15  →  [Data ingestion] Validate gauge coverage  
github.com/FlowWest/my-repo/issues/23  →  [Milestones] Draft report (subitem: Write chapter 1)
```

The preferred pattern is one GitHub issue per Monday item or subitem (1:1). A single issue can map to multiple items when it's genuinely cross-cutting, but this should be the exception — it means multiple Monday items update on every status change for that issue. Multiple issues should never map to the same Monday item; use subitems instead.

If Monday items don't exist yet, ask the user to go back to Phase 3 or create items manually first.

### 5c. Write links

After the user approves the mapping, use `change_item_column_values` to write each issue URL into the GitHub Issue link column on the correct item or subitem. See `references/board-schema.md` for the link column value format.

Confirm how many links were written when done.

---

## Phase 6: Backfill

This one-time step syncs current GitHub issue state (open/closed, assignees) into Monday without needing any issue activity to fire the workflow. It's the right way to initialize status and owner on a board that already has issue links.

Direct the user to:
1. Go to the project repo → Actions → "Backfill monday.com from GitHub issues"
2. Click Run workflow → Run workflow

The workflow log shows each issue processed and which Monday items were updated. Ask the user to spot-check a few items afterward to confirm Status and Owner look correct.

---

## Finishing up

Summarize what was completed and remind the user of two ongoing maintenance tasks:

1. **New issues going forward**: Paste the GitHub issue URL into the GitHub Issue column on the relevant Monday item. The sync workflow handles Status and Owner automatically from that point.
2. **Team roster changes**: The GitHub login → Monday display name mapping is in `FlowWest/infra/.github/workflows/` in the `USER_MAP` dict in both workflow files. Update it when team members join or leave.

---

## Reference files

- `references/board-schema.md` — column IDs, status index values, variable mapping, MCP value formats, and the `duplicate_board` mutation. Read this during Phase 2 and Phase 4.
