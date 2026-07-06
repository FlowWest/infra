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

The letter is intentionally brief. The table descriptions are plain line items — **no bullet characters** — matching the style of the reference examples.

---

## Step 1: Gather inputs

Tell the PM:

> "I'll generate your monthly progress report letter. Please upload:
> 1. **Factor time tracking report** — `time_tracking_details_MM_YYYY.pdf` (from Factor → Reports → Printable Reports → Time Detail for Export, filtered to the billing month and project)
> 2. **Draft invoice PDF**
>
> Also:
> - Were any deliverables **completed** this month, or are any **upcoming** next month? (Say 'none' to skip.)
> - Confirm your **full name and title** for the sign-off (e.g., 'Sadie Gill, Principal Data Scientist').
> - **Signature image:** Upload a `.png` or `.jpg` of your signature to include it in the sign-off. Skip to leave no gap between 'Sincerely,' and your name."

After receiving these, proceed to parse both PDFs. Then — before generating the docx — confirm the recipient and show a draft of the work descriptions for PM review.

---

## Step 2: Parse the Factor PDF

The Factor report has columns: `Date | Type | Number | Project Name | Client Name | Phase | Employee | Role | Hours | Value | Entry Description`

**Keep only `Type == "Project"` rows.** Exclude everything else: PTO, Holiday, Part-Time Staff Holiday, State Paid/Unpaid Family Leave, Unpaid Time Off, or any other non-project type.

**Group by top-level task** from the `Phase` column. The Phase looks like `"2 - Shiny Application Development/2.1 Document That Describes App Components & Functionality"` — take everything before the first `/`.

**Synthesize work descriptions:** For each top-level task, write **3–5 concise plain-text lines** summarizing the actual work done. These are NOT bullets — they are short phrases or sentences, one per line, like a list without list markers. Write in past tense. Combine similar entries across staff. Keep the language professional but plain — aim for the brevity of the reference examples, not exhaustive detail.

**Reference example (June 2026, Task 2):**
```
Continued development and iterative review of the design document, including internal reviews, refinement following a meeting with DWR, and incorporation of statistical analysis approaches.
Database schema design and updates to support application data structures.
CDEC data exploration and cleaning of data across targeted sonde stations.
Exploratory data analysis (EDA) for anomaly detection modeling.
```

Tasks with no billable entries get: `No work completed under this task.` (italic in the final doc).

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

## Step 4: Confirm recipient + show draft for PM review

**First, confirm the recipient:**
> "I'll address the letter to **[Candidate Name]** ([email]) at [org]. Is that correct, or is the Task Order Manager someone different? If different, provide their full name, title, org, address, phone, and email."

**Then, show the draft work descriptions** and ask for feedback before generating the docx:

> "Here's a draft of the work summary. Please review and let me know what to add, remove, or rephrase before I create the document.
>
> ---
> **Task 1 — Project Management**
> [line 1]
> [line 2]
>
> **Task 2 — [Task Name]**
> [line 1]
> [line 2]
> [line 3]
>
> **Task 3 — Reporting**
> No work completed under this task.
>
> **Budget and Schedule**
> Remaining balance is $[amount]
> ---"

Wait for PM feedback. Apply any edits, then generate the docx.

---

## Step 5: Generate the .docx

Read the docx skill at `/var/folders/d9/m69w1tsx69b239mkpsjjb1qh0000gn/T/claude-hostloop-plugins/0e251125660eacd6/skills/docx/SKILL.md` for generation instructions. Write and run a Python script using `python-docx`.

### Exact document structure

**1. Header — two-column invisible table** (logo right-aligned + FlowWest address):

Column widths in **twips** (NOT inches/EMUs): 3.5" = 5040 twips, 3.0" = 4320 twips.
Use `set_table_width` and `zero_cell_right_margin` helpers (below) to prevent bleed and remove logo padding.

```python
HCOL_LEFT  = int(3.5 * 1440)   # 5040 twips
HCOL_RIGHT = int(3.0 * 1440)   # 4320 twips

htable = doc.add_table(rows=1, cols=2)
remove_table_borders(htable)
set_table_width(htable, HCOL_LEFT + HCOL_RIGHT)   # 9360 twips, fixed layout

right_cell = htable.rows[0].cells[1]
zero_cell_right_margin(right_cell)   # remove unwanted right padding from logo

lp = right_cell.paragraphs[0]
lp.alignment = WD_ALIGN_PARAGRAPH.RIGHT
no_space(lp)
lp.add_run().add_picture(logo_path, width=Inches(1.75))

ap = right_cell.add_paragraph("P.O. Box 29392\nOakland, CA 94604")
ap.alignment = WD_ALIGN_PARAGRAPH.RIGHT
no_space(ap)
for r in ap.runs: r.font.size = Pt(9)

# Set column widths via tblGrid AFTER content is added
set_table_col_widths(htable, [HCOL_LEFT, HCOL_RIGHT])
```

Helpers:
```python
def remove_table_borders(table):
    tbl = table._tbl
    tblPr = tbl.find(qn('w:tblPr'))
    if tblPr is None:
        tblPr = OxmlElement('w:tblPr')
        tbl.insert(0, tblPr)
    tblBorders = OxmlElement('w:tblBorders')
    for side in ('top','left','bottom','right','insideH','insideV'):
        el = OxmlElement(f'w:{side}')
        el.set(qn('w:val'), 'none')
        tblBorders.append(el)
    tblPr.append(tblBorders)

def set_table_width(table, width_twips):
    """Fix entire table to an explicit width and set layout=fixed (prevents margin bleed)."""
    tbl = table._tbl
    tblPr = tbl.find(qn('w:tblPr'))
    if tblPr is None:
        tblPr = OxmlElement('w:tblPr')
        tbl.insert(0, tblPr)
    for old in tblPr.findall(qn('w:tblW')): tblPr.remove(old)
    tblW = OxmlElement('w:tblW')
    tblW.set(qn('w:w'), str(width_twips))
    tblW.set(qn('w:type'), 'dxa')
    tblPr.append(tblW)
    for old in tblPr.findall(qn('w:tblLayout')): tblPr.remove(old)
    tblLayout = OxmlElement('w:tblLayout')
    tblLayout.set(qn('w:type'), 'fixed')
    tblPr.append(tblLayout)

def zero_cell_right_margin(cell):
    """Remove right padding from a table cell (eliminates unwanted logo padding)."""
    tc = cell._tc
    tcPr = tc.find(qn('w:tcPr'))
    if tcPr is None:
        tcPr = OxmlElement('w:tcPr')
        tc.insert(0, tcPr)
    tcMar = tcPr.find(qn('w:tcMar'))
    if tcMar is None:
        tcMar = OxmlElement('w:tcMar')
        tcPr.append(tcMar)
    right = tcMar.find(qn('w:right'))
    if right is None:
        right = OxmlElement('w:right')
        tcMar.append(right)
    right.set(qn('w:w'), '0')
    right.set(qn('w:type'), 'dxa')
```

**2. Title + horizontal rule:**
```python
# Small gap, then bold title
doc.add_paragraph()
tp = doc.add_paragraph()
tr = tp.add_run("Monthly Progress Report")
tr.bold = True; tr.font.size = Pt(13)

# Horizontal rule via paragraph bottom border
def add_horizontal_rule(doc):
    p = doc.add_paragraph()
    pPr = p._p.get_or_add_pPr()
    pBdr = OxmlElement('w:pBdr')
    bottom = OxmlElement('w:bottom')
    bottom.set(qn('w:val'), 'single')
    bottom.set(qn('w:sz'), '6')
    bottom.set(qn('w:space'), '1')
    bottom.set(qn('w:color'), 'auto')
    pBdr.append(bottom)
    pPr.append(pBdr)
add_horizontal_rule(doc)
doc.add_paragraph()   # blank line after rule
```

**3. Recipient contact block** (plain text, 0pt spacing):
```
[Recipient Full Name]
[Recipient Title]
[Organization Name]
[Street Address]
[City, State, ZIP]
[Phone]
[Email]
```
Then **one blank line**, then:
```
Contract No: [Contract No.]
[TO Name]
(FlowWest Project No.: [Project Number])
```

**4. Date** — blank line, then `[Month spelled out] [Day], [Year]` (use invoice date)

**5. Salutation** — `Dear [Recipient Full Name],`

**6. Body paragraph 1:**
`Enclosed please find FlowWest's invoice for work completed by the project team on the [TO Name] through [Month Year].`

If PM provided deliverable info, add as a **separate paragraph** (no blank line before it):
- Completed: `The [deliverable name] was completed and delivered to ICF and the Client by [date] as planned.`
- Upcoming: `The [deliverable name] will be delivered by [date] to ICF and the Client for review.`
- None: omit.

**7. Body paragraph 2:** `Work completed during this billing period is summarized below.` — **no blank line** between this and paragraph 1 (or the deliverable paragraph), and **no blank line** before the table.

**8. Task table** — immediately after paragraph 2 (no blank paragraph between).

**9. Body paragraph 3** (after table): `Thank you for the continuing opportunity to work on this project. If you have any questions or comments regarding these charges, please do not hesitate to contact me.`

**10. Sign-off:**
```python
doc.add_paragraph("Sincerely,")

if sig_path and os.path.exists(sig_path):
    doc.add_paragraph()   # blank line before image
    sp = doc.add_paragraph()
    sp.add_run().add_picture(sig_path, width=Inches(2))
    doc.add_paragraph()   # blank line after image
# no blank line if no signature — name follows directly

p_name = doc.add_paragraph(pm_name)
no_space(p_name)
p_title = doc.add_paragraph(pm_title)
no_space(p_title)
p_org = doc.add_paragraph("FlowWest, LLC")
no_space(p_org)
```

---

### Task table (exact formatting)

Column widths at **1:3 ratio** — total 6.5" (8.5" minus 1" margins each side):
- Task column: **2340 twips** (1.625")
- Description column: **7020 twips** (4.875")

Set widths via `tblGrid` XML **after** all rows are added (python-docx cell width alone is unreliable — always set the grid):

```python
TASK_TW = 2340   # twips
DESC_TW = 7020

table = doc.add_table(rows=1, cols=2)
table.style = 'Table Grid'

# Header row
hdr = table.rows[0].cells
hdr[0].paragraphs[0].clear(); hdr[0].paragraphs[0].add_run("Task").bold = True
hdr[1].paragraphs[0].clear(); hdr[1].paragraphs[0].add_run("Description").bold = True

for task_num, task_name, lines in task_data:
    row = table.add_row().cells
    # Left: task number, bold, left-aligned
    p0 = row[0].paragraphs[0]; p0.clear()
    p0.add_run(str(task_num)).bold = True
    # Right: task name bold, then bullet items
    p1 = row[1].paragraphs[0]; p1.clear()
    p1.add_run(task_name).bold = True
    if lines:
        for line in lines:
            p1.add_run('\n• ' + line)
    else:
        p1.add_run('\n')
        p1.add_run('No work completed under this task.').italic = True

# Budget and Schedule — merged row
brow = table.add_row().cells
brow[0].merge(brow[1])
p = brow[0].paragraphs[0]; p.clear()
p.add_run('Budget and Schedule').bold = True
p.add_run('\n')
p.add_run(f'Remaining balance is ${remaining_balance}').italic = True

# Apply column widths via tblGrid AFTER all rows added
set_table_col_widths(table, [TASK_TW, DESC_TW])
# Fix table to body width — prevents bleed into margins
set_table_width(table, TASK_TW + DESC_TW)   # 9360 twips = 6.5"
```

**Page setup:** 8.5" × 11", all margins 1.0"

---

## Step 6: Deliver

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
