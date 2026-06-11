# FlowWest Shared GitHub Actions Infrastructure

This repository is **public**. Do not commit credentials, tokens, API keys, board IDs, column IDs, or any project-specific identifiers here. All sensitive or project-specific values belong in secrets set on individual project repos.

## What's here

```
.github/workflows/          Reusable workflows — these run when called by other repos
templates/                  Caller files — copy these into project repos to wire up a workflow
```

---

## Workflows

### `sync-issue-to-monday.yml`

Syncs GitHub issue lifecycle events to a monday.com board. Fires automatically on issue activity and updates Status and Owner on the matching monday item or subitem.

| GitHub event | monday update |
|---|---|
| Issue closed | Status → Done |
| Issue reopened | Status → In Progress |
| Label `blocked` added | Status → Stuck |
| Label `blocked` removed | Status → In Progress |
| Issue assigned | Owner set to assignee |
| Issue unassigned | Owner cleared |

### `backfill-monday-sync.yml`

One-time manual sync that reads the current state of every issue in the repo and updates all linked monday items accordingly. Run this when first setting up a project board after pasting issue URLs into monday — it sets Status and Owner to match GitHub without needing to touch any issues.

| GitHub issue state | monday update |
|---|---|
| Open | Status → In Progress |
| Closed | Status → Done |
| Has assignees | Owner set to assignees |
| No assignees | Owner cleared |

**Maintaining the team roster**

The mapping of GitHub logins to monday display names is in the `USER_MAP` dict in both workflow files. Update it when team members join or leave — one change here applies to all connected repos.

---

## Setting up a new project repo

### 1. Copy both caller templates

Copy both files from `templates/` into the project repo under `.github/workflows/`:

```
templates/sync-issue-to-monday.yml   →   .github/workflows/sync-issue-to-monday.yml
templates/backfill-monday-sync.yml   →   .github/workflows/backfill-monday-sync.yml
```

No edits to either file are needed.

### 2. Add secrets to the project repo

Go to the project repo → Settings → Secrets and variables → Actions and add each of the following. These secrets are shared by both workflows — set them once and both work.

| Secret | Description |
|---|---|
| `MONDAY_API_TOKEN` | Personal API token for the monday bot user |
| `MONDAY_BOARD_ID` | ID of the main monday board for this project |
| `MONDAY_ITEM_LINK_COL` | Column ID of the GitHub Issue link column on items |
| `MONDAY_ITEM_STATUS_COL` | Column ID of the Status column on items |
| `MONDAY_ITEM_OWNER_COL` | Column ID of the Owner column on items |
| `MONDAY_SUBITEM_BOARD_ID` | Board ID of the subitems board |
| `MONDAY_SUBITEM_LINK_COL` | Column ID of the GitHub Issue link column on subitems |
| `MONDAY_SUBITEM_STATUS_COL` | Column ID of the Status column on subitems |
| `MONDAY_SUBITEM_OWNER_COL` | Column ID of the Owner column on subitems |

### 3. Finding your IDs

**Board ID**
Open the board in monday. The board ID is the number in the URL:
`https://[org].monday.com/boards/[BOARD_ID]`

**Subitem board ID**
Subitems live on a separate board with their own ID. Click into any subitem row to expand it, then read the second board ID from the URL.

**Column IDs**
Hover over any column header → click ⋯ → Settings. The column ID is shown at the bottom of the panel (e.g. `color_mm3qjykx`, `link_mm41jw5g`). Do this for each required column on both the main items board and the subitems board.

### 4. Configure the monday board

The board must have these columns on both items and subitems:

| Column | Type | Purpose |
|---|---|---|
| GitHub Issue | Link | Paste the GitHub issue URL here to connect it |
| Status | Status | Updated automatically on issue events |
| Owner | People | Updated automatically on issue assignment |

For each issue you want to track, paste its GitHub URL into the GitHub Issue link column on the matching item or subitem. The same URL can appear on multiple items or subitems — all matches are updated when the issue changes.

### 5. Run the initial backfill

After pasting all issue URLs into monday, trigger the backfill to set Status and Owner to match current GitHub state:

1. Go to the project repo on GitHub
2. Click **Actions** → **Backfill monday.com from GitHub issues**
3. Click **Run workflow** → **Run workflow**

The workflow log shows each issue processed, which monday items were updated, and which issues had no monday link (expected for issues not tracked on the board).