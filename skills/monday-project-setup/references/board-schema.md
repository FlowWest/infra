# FlowWest Template Board Schema

Template URL: `https://flowwest-force.monday.com/template_center/template/22748268`
Template ID: `22748268`
Workspace: Data (id: `1366388`)
Subitem board ID: `18414974766`

## Main item columns

| Column title   | Column ID                  | Type       | Notes |
|----------------|----------------------------|------------|-------|
| Name           | `name`                     | name       | Always present |
| Status         | `color_mm3qjykx`           | status     | In Progress / Done / Stuck / Not Started |
| Points         | `dropdown_mm41qtet`        | dropdown   | Story points: 1, 3, 5, 8, 11 |
| Owner          | `multiple_person_mm3qka0s` | people     | |
| Start Date     | `date_mm3qds32`            | date       | |
| End Date       | `date_mm3qvs6d`            | date       | |
| Priority       | `color_mm3qf6cr`           | status     | Low / Medium / High / Critical |
| Subitems       | `subtasks_mm3qvrvy`        | subtasks   | Linked to subitem board |
| Dependencies   | `dependency_mm3q8ss6`      | dependency | |
| GitHub Issue   | `link_mm41jw5g`            | link       | Primary GitHub issue URL |
| Issue Progress | `columns_battery_mm41jzed` | progress   | Auto-calculated from subitems' Status |

## Subitem columns (board 18414974766)

| Column title | Column ID       | Type   | Notes |
|--------------|-----------------|--------|-------|
| Name         | `name`          | name   | |
| Owner        | `person`        | people | |
| Status       | `status`        | status | Working on it / Done / Stuck |
| Date         | `date0`         | date   | |
| GitHub Issue | `link_mm411dje` | link   | GitHub issue URL |

## Status index values (used by GitHub Actions)

Main items Status (`color_mm3qjykx`): `0` = In Progress, `1` = Done, `2` = Stuck

Subitems Status (`status`): `0` = Working on it, `1` = Done, `2` = Stuck

## GitHub Actions variable mapping

These env var names are what the caller workflow templates expect. After creating a new board (duplicate or blank), always call `get_board_info` to retrieve the actual IDs — they will differ from the template values below.

| Variable                  | Template value             | Description |
|---------------------------|----------------------------|-------------|
| `MONDAY_BOARD_ID`         | _(new board's ID)_         | Read from board URL |
| `MONDAY_ITEM_LINK_COL`    | `link_mm41jw5g`            | GitHub Issue link column on items |
| `MONDAY_ITEM_STATUS_COL`  | `color_mm3qjykx`           | Status column on items |
| `MONDAY_ITEM_OWNER_COL`   | `multiple_person_mm3qka0s` | Owner column on items |
| `MONDAY_SUBITEM_BOARD_ID` | `18414974766`              | Read from subitem column settings |
| `MONDAY_SUBITEM_LINK_COL` | `link_mm411dje`            | GitHub Issue link column on subitems |
| `MONDAY_SUBITEM_STATUS_COL` | `status`                 | Status column on subitems |
| `MONDAY_SUBITEM_OWNER_COL` | `person`                  | Owner column on subitems |

## MCP column value formats

**Link column** (GitHub Issue):
```json
{"url": "https://github.com/FlowWest/repo/issues/1", "text": "Issue #1"}
```

**Status column:**
```json
{"index": 0}
```

## `create_board` from template mutation

```graphql
mutation {
  create_board(
    board_name: "PROJECT NAME HERE",
    board_kind: public,
    workspace_id: 1366388,
    template_id: 22748268
  ) {
    id name url
  }
}
```

Use this via the `all_monday_api` tool. After creation, call `get_board_info` to retrieve the actual column IDs — they may differ from the template values listed above.
