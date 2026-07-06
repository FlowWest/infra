---
name: monthly-progress-report
description: Generate a FlowWest monthly invoice progress report letter as a .docx file. Use this skill whenever a PM wants to create a monthly progress report, invoice summary letter, or billing period summary for a client. Triggers on phrases like "progress report", "monthly report", "invoice letter", "progress report letter", "billing summary", or when a user uploads a Factor time tracking PDF and draft invoice and wants to summarize the month's work. Always use this skill when the user mentions generating a report to accompany an invoice, even if they don't say "progress report" explicitly.
---

# Monthly Progress Report Generator

Generates a formal Word document (.docx) progress report letter for FlowWest project managers to accompany monthly client invoices. Follows the exact format and length of FlowWest's established letter template (simple letter style — not a memo).

## What you'll produce

A clean, concise .docx letter containing:
- FlowWest logo at top
- Recipient contact block + contract details
- Date, salutation, 1–2 sentence intro
- Optional deliverable status sentence
- Task summary table (task number | plain-text description)
- Merged "Budget and Schedule" row with remaining balance
- Thank-you paragraph, sign-off

The letter is intentionally brief. Descriptions are concise bullet items per task.

---

## Step 1: Gather inputs

Tell the PM:

> "I'll generate your monthly progress report letter. Please upload:
> 1. **Factor time tracking report** — `time_tracking_details_MM_YYYY.pdf` (from Factor → Reports → Printable Reports → Time Detail for Export, filtered to the billing month and project)
> 2. **Draft invoice PDF**
>
> Also:
> - **GitHub repos** *(optional)* — If this project has a GitHub repo, provide the repo(s) (e.g. `FlowWest/my-repo`) and a GitHub Personal Access Token (PAT) — a token from GitHub Settings → Developer settings → Personal access tokens, with `repo` scope. Commits will be used to enrich the work descriptions. Skip if not applicable.
> - Were any deliverables **completed** this month, or are any **upcoming** next month? (Say 'none' to skip.)
> - Confirm your **full name and title** for the sign-off (e.g., 'Sadie Gill, Principal Data Scientist').
> - **Signature image:** Upload a `.png` or `.jpg` of your signature to include it in the sign-off. Skip to leave no gap between 'Sincerely,' and your name."

After receiving these, proceed. Then — before generating the docx — confirm the recipient and show a draft of the work descriptions for PM review.

---

## Step 2: Parse the Factor PDF

The Factor report has columns: `Date | Type | Number | Project Name | Client Name | Phase | Employee | Role | Hours | Value | Entry Description`

**Keep only `Type == "Project"` rows.** Exclude everything else: PTO, Holiday, Part-Time Staff Holiday, State Paid/Unpaid Family Leave, Unpaid Time Off, or any other non-project type.

**Group by top-level task** from the `Phase` column. The Phase looks like `"2 - Shiny Application Development/2.1 Document That Describes App Components & Functionality"` — take everything before the first `/`.

For each top-level task, collect all the `Entry Description` values from its rows. These are the raw inputs for synthesis in Step 4.

Tasks with no billable entries get: `No work completed under this task.` (italic in the final doc).

---

## Step 2b: Fetch GitHub commits *(skip if no repos provided)*

Use Python via bash. For each repo:

1. Fetch the default branch name: `GET /repos/{owner}/{repo}`
2. Fetch all branches: `GET /repos/{owner}/{repo}/branches?per_page=100`
3. Pull commits from the default branch for the billing period: `GET /repos/{owner}/{repo}/commits?sha={default_branch}&since=...&until=...&per_page=100`
4. For feature branches, use the compare endpoint to get only commits not yet in the default branch, then filter by date
5. Auth header: `Authorization: Bearer <token>`

Strip merge commits. Deduplicate by SHA. The result is a flat list of commit messages per repo, used in Step 4.

---

## Step 3: Parse the draft invoice PDF

Extract:
- **Candidate recipient**: from the `cc:` line (name and email). If only an email is present, derive the name from it or flag for PM confirmation.
- **Recipient org / address / phone / email**: from the "INVOICE FOR" block
- **Project name**, **FlowWest project number** (e.g., "028-11")
- **Contract No.**, **TO No.**, **TO Name**
- **Billing period** (e.g., "6/1/2026 to 6/30/2026") and **invoice date**
- **Remaining balance**: the "Budget Remaining" total from the invoice summary row
- **All top-level task names**: every top-level task row in the invoice table

---

## Step 4: Synthesize work descriptions

For each top-level task, write **3–5 concise bullet items** summarizing what was done. Past tense, plain language — outcomes and features, not implementation details.

Use whatever sources are available:
- **Factor only**: synthesize from the `Entry Description` values
- **Factor + GitHub**: use both — Factor anchors which tasks had billable hours, commits enrich the narrative. Map commits to tasks by matching commit content to task names.
- **No Factor entries for a task but relevant commits exist**: use commits rather than marking it "No work completed"

**Reference example:**
```
• Continued development and iterative review of the design document.
• Database schema design and updates to support application data structures.
• CDEC data exploration and cleaning across targeted sonde stations.
• Exploratory data analysis for anomaly detection modeling.
```

---

## Step 5: Confirm recipient + show draft for PM review

**First, confirm the recipient:**
> "I'll address the letter to **[Candidate Name]** ([email]) at [org]. Is that correct, or is the Task Order Manager someone different? If different, provide their full name, title, org, address, phone, and email."

**Then, show the draft work descriptions** and ask for feedback before generating the docx:

> "Here's a draft of the work summary. Please review and let me know what to add, remove, or rephrase before I create the document.
>
> ---
> **Task 1 — Project Management**
> • [line 1]
> • [line 2]
>
> **Task 2 — [Task Name]**
> • [line 1]
> • [line 2]
> • [line 3]
>
> **Task 3 — Reporting**
> No work completed under this task.
>
> **Budget and Schedule**
> Remaining balance is $[amount]
> ---"

Wait for PM feedback. Apply any edits, then generate the docx.

---

## Step 6: Generate the .docx

Use the bundled script at `scripts/generate_report.py`. Construct a JSON input file from everything gathered above, then run the script.

```json
{
  "output_path": "/abs/path/to/[Month]_[Year]_[ProjectNo]_Progress_Report.docx",
  "logo_path": "/Users/sadiegill/Documents/infra/skills/monthly-progress-report/references/flowwest_logo.jpg",
  "sig_path": "/path/to/uploaded/signature.png",
  "recipient": {
    "name": "[Full Name]",
    "title": "[Title]",
    "org": "[Organization]",
    "address": "[Street]",
    "city_state_zip": "[City, State ZIP]",
    "phone": "[Phone]",
    "email": "[Email]"
  },
  "contract": {
    "contract_no": "[Contract No.]",
    "to_name": "[TO Name]",
    "project_no": "[Project No.]"
  },
  "date": "[Month Day, Year]",
  "intro": "Enclosed please find FlowWest's invoice for work completed by the project team on the [TO Name] through [Month Year].",
  "deliverable": null,
  "tasks": [
    { "num": "1", "name": "Project Management", "lines": ["bullet 1", "bullet 2"] },
    { "num": "2", "name": "[Task Name]",         "lines": [] }
  ],
  "remaining_balance": "[amount]",
  "pm_name": "[PM Full Name]",
  "pm_title": "[PM Title]"
}
```

Omit `sig_path` (or set to `null`) if no signature was uploaded. The `deliverable` field takes a full sentence or `null`. Empty `lines` arrays produce an italic "No work completed" line.

Run the script:
```bash
python3 scripts/generate_report.py /tmp/report_input.json
```

---

## Step 7: Deliver

1. Save the .docx to the workspace folder as `[Month]_[Year]_[ProjectNumber]_Progress_Report.docx`
2. Present with `present_files`
3. Add: "This is a draft — review before sending."

---

## Logo note

The FlowWest logo is stored at:
`/Users/sadiegill/Documents/infra/skills/monthly-progress-report/references/flowwest_logo.jpg`

If that file doesn't exist when the skill runs, skip the logo silently, generate the document without it, and tell the PM: "Logo file not found at references/flowwest_logo.jpg — add it there to include it automatically in future runs."

---

## Signature note

Use the signature image only if the PM uploaded one in Step 1. Set `sig_path` to the uploaded file path, or `None` if they skipped it. No fallback — do not look anywhere else.
