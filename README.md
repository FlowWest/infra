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

Syncs GitHub issue lifecycle events to a monday.com board. Handles status and owner updates for both items and subitems. All board configuration is passed in by the calling repo — nothing project-specific lives in this file.

| GitHub event | monday update |
|---|---|
| Issue closed | Status → Done |
| Issue reopened | Status → In Progress |
| Label `blocked` added | Status → Stuck |
| Label `blocked` removed | Status → In Progress |
| Issue assigned | Owner set to assignee |
| Issue unassigned | Owner cleared |

**Maintaining the team roster**

The mapping of GitHub logins to monday display names is in the `user_map` dict in `.github/workflows/sync-issue-to-monday.yml`. Update it when team members join or leave — one change here applies to all connected repos.

---

## Setting up a new project repo

### 1. Copy the caller template

Copy `templates/sync-issue-to-monday.yml` into the project repo at:

```
.github/workflows/sync-issue-to-monday.yml
```

No edits to the file are needed.

### 2. Add secrets to the project repo

Go to the project repo → Settings → Secrets and variables → Actions and add each of the following:

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