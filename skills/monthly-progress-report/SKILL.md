---
name: monthly-progress-report
description: >
  Generate a FlowWest monthly invoice progress report letter as a .docx file.
  Use this skill whenever a PM wants to create a monthly progress report, invoice
  summary letter, or billing period summary for a client. Always use this skill
  when a user uploads a Factor time tracking PDF, a draft invoice, or mentions
  generating a report to accompany an invoice — even if they don't say "progress
  report" explicitly. Triggers on: "progress report", "monthly report", "invoice
  letter", "billing summary", "write up my hours", "summarize this month's work".
---

# Monthly Progress Report Generator

Generates a formal Word document (.docx) progress report letter for FlowWest project managers to accompany monthly client invoices. Simple letter style — not a memo. Intentionally brief: 3–5 bullet items per task, plain language, no implementation details.

---

## Step 1: Gather inputs

Ask the PM for:

1. **Factor time tracking report** (required) — `time_tracking_details_MM_YYYY.pdf`, exported from Factor → Reports → Printable Reports → Time Detail for Export, filtered to the billing month and project
2. **Draft invoice PDF** (required)
3. **GitHub repos** (optional) — repo(s) in `Owner/repo` format (e.g. `FlowWest/my-repo`) plus a GitHub Personal Access Token (PAT, from GitHub Settings → Developer settings → Personal access tokens with `repo` scope). Commits will enrich the work descriptions.
4. **Deliverables** (optional) — any deliverables completed or due next month; say 'none' to skip
5. **PM full name and title** — for the sign-off
6. **Signature image** (optional) — upload a `.png` or `.jpg`; skip to have the name follow directly after "Sincerely,"

---

## Step 2: Parse the Factor PDF

The Factor report columns are: `Date | Type | Number | Project Name | Client Name | Phase | Employee | Role | Hours | Value | Entry Description`

Keep only rows where `Type == "Project"` — discard PTO, Holiday, Part-Time Staff Holiday, leave types, and anything else non-project.

Group by top-level task from the `Phase` column. Phase looks like `"2 - Shiny Application Development/2.1 Document..."` — take everything before the first `/`.

Collect all `Entry Description` values per top-level task. These feed into Step 5 synthesis. Tasks with no billable rows will show "No work completed under this task." (italic) in the final document.

---

## Step 3: Fetch GitHub commits *(skip if no repos provided)*

For each repo, run a Python script via bash:

1. Get the default branch: `GET https://api.github.com/repos/{owner}/{repo}`
2. Get all branches: `GET https://api.github.com/repos/{owner}/{repo}/branches?per_page=100`
3. Fetch commits on the default branch for the billing period using ISO 8601 dates from the invoice (e.g. `since=2026-06-01T00:00:00Z&until=2026-06-30T23:59:59Z`):
   `GET https://api.github.com/repos/{owner}/{repo}/commits?sha={default_branch}&since=...&until=...&per_page=100`
4. For each feature branch (anything other than the default, `staging`, `production`, `prod`, `build-staging`), get commits unique to that branch via the compare endpoint, then filter to the billing period by `commit.author.date`:
   `GET https://api.github.com/repos/{owner}/{repo}/compare/{default_branch}...{feature_branch}`
5. Add `Authorization: Bearer <token>` to every request. Skip branches returning 404 or 500.

Strip any commit message starting with `"Merge pull request"` or `"Merge branch"`. Deduplicate by SHA. Output: a flat list of commit messages grouped by repo, used in Step 5.

---

## Step 4: Parse the draft invoice PDF

Extract:
- **Recipient candidate**: the `cc:` line (name + email). If only an email, derive the name or flag for PM confirmation.
- **Recipient org / address / phone / email**: from the "INVOICE FOR" block
- **Project name**, **FlowWest project number** (e.g. `028-11`)
- **Contract No.**, **TO No.**, **TO Name**
- **Billing period** and **invoice date**
- **Remaining balance**: the "Budget Remaining" total
- **All top-level task names** from the invoice table

---

## Step 5: Synthesize work descriptions

For each top-level task, write 3–5 concise bullet items. Past tense, plain language — describe outcomes and features, not implementation details (no file names, library names, or variable names).

Draw from available sources:
- **Factor only**: synthesize from the `Entry Description` values
- **Factor + GitHub**: Factor confirms which tasks had billable hours; commit messages enrich the narrative. Map commits to tasks by matching their content to task names.
- **Task has commits but no Factor hours**: use commits rather than "No work completed"

Reference example:
```
• Continued development and iterative review of the design document.
• Database schema design and updates to support application data structures.
• CDEC data exploration and cleaning across targeted sonde stations.
• Exploratory data analysis for anomaly detection modeling.
```

---

## Step 6: Confirm recipient + PM review

Confirm the recipient first:
> "I'll address the letter to **[Name]** ([email]) at [org]. Correct, or is there a different Task Order Manager?"

Then show the draft and ask for feedback:

> **Task 1 — Project Management**
> • …
>
> **Task 2 — [Name]**
> • …
>
> **Task 3 — Reporting**
> No work completed under this task.
>
> **Budget and Schedule**
> Remaining balance is $[amount]

Wait for PM feedback. Apply edits, then proceed.

---

## Step 7: Generate the .docx

Use the bundled script at `scripts/generate_report.py` (path relative to the skill root — the directory containing this SKILL.md). Write the JSON input to a temp file and run:

```bash
python3 /path/to/skill-root/scripts/generate_report.py /tmp/report_input.json
```

JSON structure:
```json
{
  "output_path": "/abs/path/to/workspace/[Month]_[Year]_[ProjectNo]_Progress_Report.docx",
  "logo_path": "references/flowwest_logo.jpg",
  "sig_path": "/abs/path/to/uploaded/signature.png",
  "recipient": {
    "name": "",
    "title": "",
    "org": "",
    "address": "",
    "city_state_zip": "",
    "phone": "",
    "email": ""
  },
  "contract": {
    "contract_no": "",
    "to_name": "",
    "project_no": ""
  },
  "date": "[Month Day, Year]",
  "intro": "Enclosed please find FlowWest's invoice for work completed by the project team on the [TO Name] through [Month Year].",
  "deliverable": null,
  "tasks": [
    { "num": "1", "name": "Project Management", "lines": ["bullet 1", "bullet 2"] },
    { "num": "2", "name": "[Task Name]", "lines": [] }
  ],
  "remaining_balance": "[amount]",
  "pm_name": "",
  "pm_title": ""
}
```

Notes:
- `logo_path` is relative to the skill root and resolves automatically; if the file is missing the logo is silently skipped
- Set `sig_path` to the uploaded file's absolute path, or omit/`null` for no signature
- `deliverable` is a full sentence or `null`
- Empty `lines` arrays produce italic "No work completed under this task."

---

## Step 8: Deliver

Present the .docx with `present_files` and say: "This is a draft — review before sending."
