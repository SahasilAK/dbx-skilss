---
name: attribute-mapper
description: >
  Use this skill when a user wants to map, trace, or document gold-layer attributes
  from SAS source scripts through to Databricks column names with full lineage and
  transformation details. Triggers include: "map my gold attributes", "trace SAS lineage
  to Databricks", "build an attribute mapping spreadsheet", "cross-reference my column
  mappings", "generate a gold-layer mapping file", or any request that involves reading
  knowledge base MD files and producing a structured Excel mapping output. ALWAYS
  trigger this skill — do not attempt attribute mapping without it.
---

## Changelog
| Version | Date | Summary |
|---|---|---|
| v1 | initial | Base release |

# Attribute Mapper — Operational Instructions

## Purpose

This skill turns Claude into a Master Data / Attribute Mapper. It reads SAS-derived
knowledge base Markdown files, cross-references three user-supplied Excel mapping sheets,
and produces one structured gold-layer attribute mapping `.xlsx` output file.

> **⚠️ Hard constraint:** Every table name, column name, and transformation recorded in
> the output MUST be confirmed from the knowledge base files or the mapping Excel sheets.
> NEVER invent, infer from naming conventions, or fill gaps with assumptions. When a
> value cannot be confirmed, trigger the **Ambiguity Protocol**.

---

## Step 1 — Collect All Inputs

Before doing anything else, confirm that all required inputs have been provided.
If any required input is missing, ask for it before proceeding.

```
Required inputs:
  1. knowledge_base_path  — folder of .md files from sas-script-analyzer (or similar)
  2. table_mapping_excel  — Excel: SAS source table → Databricks table name
  3. column_mapping_excel — Excel: SAS source column → Databricks column name (scoped by Databricks table)
  4. gold_attributes_excel — Excel: gold attribute name + expected data type (master list)

Optional input:
  5. catalog_path — Databricks catalog path for live data type lookup
                    (if omitted, databricks_datatype column will be left blank)
```

Present this checklist to the user and get confirmation before loading any files.

---

## Step 2 — Pre-Flight: Load and Index All Reference Data

Run all loading steps in order. Stop and report any failure immediately.

### 2a. Load the Gold Attributes List

```python
import pandas as pd

gold = pd.read_excel(gold_attributes_excel, dtype=str).fillna('')
# Expected columns: field_name, data_type
# Verify both columns are present — if not, trigger Ambiguity Protocol
```

Print column headers to the user and confirm the expected column names before proceeding.

### 2b. Load the Table Mapping

```python
tbl_map = pd.read_excel(table_mapping_excel, dtype=str).fillna('')
# Expected columns: sas_source_table, databricks_table_name
# Normalize to lowercase, strip whitespace for all lookups
tbl_map['sas_source_table']       = tbl_map['sas_source_table'].str.strip().str.lower()
tbl_map['databricks_table_name']  = tbl_map['databricks_table_name'].str.strip().str.lower()
```

### 2c. Load the Column Mapping

```python
col_map = pd.read_excel(column_mapping_excel, dtype=str).fillna('')
# Expected columns: databricks_table_name, sas_source_column, databricks_column_name
# Normalize all three
for c in ['databricks_table_name','sas_source_column','databricks_column_name']:
    col_map[c] = col_map[c].str.strip().str.lower()
```

### 2d. Index the Knowledge Base

```python
import os, pathlib

kb_files = sorted(pathlib.Path(knowledge_base_path).glob('*.md'))
# Build a dict: {filename_stem: full_text}
kb = {f.stem: f.read_text(encoding='utf-8') for f in kb_files}
```

Print the count of loaded files in each category to the user as a pre-flight summary:
```
Pre-flight complete
───────────────────────────────────────────────────
Gold attributes     : N rows
Table mappings      : N rows
Column mappings     : N rows
Knowledge base files: N .md files
catalog_path        : [provided / not provided]
```

---

## Step 3 — Lineage Tracing (Per Gold Attribute)

Process each row in the gold attributes list **one at a time** in sequence.
For each `field_name`, execute all five sub-steps. Do not skip any step.

### Sub-step A — Search the Knowledge Base

Search every `.md` file in `kb` for the `field_name` value (case-insensitive).
Look in:
- Section 3 (Attribute Inventory) — "Output Column" column
- Section 6 (Cross-Script Lineage) — "Attribute" column
- Section 4 (Transformation Logic Blocks) — narrative text

Extract from the first confirmed match:
- `sas_source_table` — the real table name from LIBNAME / FROM / SET (not a SAS alias)
- `sas_source_column` — the original source column (before any rename or alias)
- `transformation` — any logic applied: CASE WHEN, calculation, function, RETAIN, ARRAY
- `join_condition` — any JOIN keys and join type (INNER, LEFT, etc.)
- `conditional_logic` — any WHERE, IF/THEN/ELSE, or CASE conditions

If the attribute appears in **Section 6 (Cross-Script Lineage)**, follow the lineage path
back through all intermediate TRN scripts until you reach the EXT reference or literal.
Record the full chain in `transformation`.

If the attribute is **not found** in any `.md` file: set `verified_logic = "No"`,
record `sas_source_table = "[NOT FOUND IN KB]"` and add a note. Continue to Sub-step E.

### Sub-step B — Table Mapping Lookup

Look up `sas_source_table` (lowercased, stripped) in `tbl_map['sas_source_table']`.

```python
match = tbl_map[tbl_map['sas_source_table'] == sas_source_table.lower().strip()]
databricks_table_name = match['databricks_table_name'].values[0] if len(match) else None
```

If no match: set `verified_logic = "No"`, record `databricks_table_name = "[NOT IN TABLE MAP]"`, add note.

### Sub-step C — Column Mapping Lookup

Using the `databricks_table_name` from Sub-step B, look up `sas_source_column` in `col_map`:

```python
hits = col_map[
    (col_map['databricks_table_name'] == databricks_table_name) &
    (col_map['sas_source_column'] == sas_source_column.lower().strip())
]
databricks_column_name = ', '.join(hits['databricks_column_name'].tolist()) if len(hits) else None
```

**Multi-source rule:** If the gold attribute maps to more than one source column, collect
ALL matching rows. Store all `databricks_column_name` values comma-separated. Store all
`sas_source_column` values comma-separated in `sas_source_column`.

If no match: set `verified_logic = "No"`, record `databricks_column_name = "[NOT IN COLUMN MAP]"`, add note.

### Sub-step D — Databricks Data Type (Optional)

Only if `catalog_path` was provided:

```bash
# Read the column schema from the catalog path
# Adapt to the actual catalog format (Delta metadata, JSON schema, etc.)
```

If `catalog_path` was not provided: leave `databricks_datatype` blank.
If provided but the column is not found: record `"[NOT FOUND IN CATALOG]"`.

### Sub-step E — Set verified_logic

```
verified_logic = "Yes"  if Sub-steps A, B, and C all succeeded without flags
verified_logic = "No"   if any sub-step failed, was flagged, or required an assumption
```

Collect all output values for this attribute into one result row (see Output Spec below).

---

## Step 4 — Build and Write the Output Excel File

After all gold attributes are processed, write the output file using `openpyxl`.

### Output Column Specification (exact order)

| # | Column Name             | Source |
|---|-------------------------|--------|
| 1 | `field_name`            | Gold attributes Excel |
| 2 | `data_type`             | Gold attributes Excel (expected type) |
| 3 | `verified_logic`        | `"Yes"` or `"No"` — see Sub-step E |
| 4 | `transformation`        | Full step-by-step logic from KB + lineage chain. For each step include: WHAT the transformation does, WHY it is done, WHEN it applies, and the **Databricks-equivalent logic** using confirmed Databricks table and column names only. |
| 5 | `join_condition`        | JOIN type, join tables, and join keys from KB |
| 6 | `conditional_logic`     | All WHERE / IF-THEN-ELSE / CASE WHEN / filter conditions |
| 7 | `databricks_table_name` | From table mapping Excel (Sub-step B) |
| 8 | `databricks_column_name`| From column mapping Excel (Sub-step C); comma-separated if multi-source |
| 9 | `databricks_datatype`   | From catalog (Sub-step D), or blank if not provided |
| 10| `sas_source_table`      | Confirmed SAS table name from KB (Sub-step A) |
| 11| `sas_source_column`     | Confirmed SAS column name from KB; comma-separated if multi-source |
| 12| `notes`                 | Any flags, `[NOT FOUND]` markers, `[UNRESOLVED]` items, or mapping caveats |

### Excel Writing Pattern

```python
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment

HEADERS = [
    'field_name','data_type','verified_logic','transformation',
    'join_condition','conditional_logic','databricks_table_name',
    'databricks_column_name','databricks_datatype',
    'sas_source_table','sas_source_column','notes'
]
WIDE_COLS  = {'transformation', 'join_condition', 'conditional_logic', 'notes'}
HEADER_FILL = PatternFill('solid', start_color='1F4E79')  # Dark blue
VERIFIED_FILL = {'Yes': PatternFill('solid', start_color='C6EFCE'),   # Green
                 'No':  PatternFill('solid', start_color='FFCCCC')}   # Red

wb = Workbook()
ws = wb.active
ws.title = 'Gold Attribute Mapping'

# Header row
for col_idx, h in enumerate(HEADERS, 1):
    cell = ws.cell(row=1, column=col_idx, value=h)
    cell.font = Font(bold=True, color='FFFFFF', name='Arial', size=10)
    cell.fill = HEADER_FILL
    cell.alignment = Alignment(horizontal='center', wrap_text=True)

# Freeze header row
ws.freeze_panes = 'A2'

# Data rows
for row_idx, record in enumerate(results, 2):
    for col_idx, key in enumerate(HEADERS, 1):
        cell = ws.cell(row=row_idx, column=col_idx, value=record.get(key, ''))
        cell.alignment = Alignment(wrap_text=True, vertical='top')
        cell.font = Font(name='Arial', size=10)
    # Color-code verified_logic column (col 3)
    v = record.get('verified_logic', '')
    if v in VERIFIED_FILL:
        ws.cell(row=row_idx, column=3).fill = VERIFIED_FILL[v]

# Column widths
for col_idx, h in enumerate(HEADERS, 1):
    ws.column_dimensions[ws.cell(1, col_idx).column_letter].width = (
        60 if h in WIDE_COLS else 25
    )

output_path = '/mnt/user-data/outputs/gold_attribute_mapping.xlsx'
wb.save(output_path)
```

> Do NOT run `scripts/recalc.py` — this file contains no formulas, only data values.
> Save and present the file directly.

---

## Guardrails

| # | NEVER DO THIS |
|---|---|
| M1 | Invent a Databricks table or column name not present in the mapping Excel files. |
| M2 | Invent a SAS source table or column name not present verbatim in the knowledge base MD. |
| M3 | Write transformation logic that is not documented in the knowledge base. |
| M4 | Set `verified_logic = "Yes"` if any of Sub-steps A, B, or C returned a flag. |
| M5 | Skip a step in the lineage chain — every TRN hop must be documented in `transformation`. |
| M6 | Assume a column from one Databricks table belongs to another to make a lookup succeed. |
| M7 | Collapse multi-source columns into one without recording all source columns in `sas_source_column`. |
| M8 | Leave `notes` blank when any flag, `[NOT FOUND]`, or `[UNRESOLVED]` tag is recorded elsewhere in the row. |
| M9 | Process the next attribute while an unresolved ambiguity for the current one is still open. |
| M10| Apply a pattern observed in one attribute's lineage to another attribute without verifying it independently. |

---

## Ambiguity Protocol

**Trigger this when:**
- The gold attribute name is not found in any knowledge base MD file.
- A SAS source table is found in the KB but absent from the table mapping Excel.
- A SAS source column is found but absent from the column mapping Excel.
- The KB shows two possible source tables or columns for one attribute.
- A transformation references a TRN script not present in the knowledge base.
- A macro resolves to a table or column name dynamically and the resolution is ambiguous.

**Format — use this exactly:**

```
⚠️  AMBIGUITY — Guidance Needed
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Attribute: [field_name]
Step     : [e.g., Sub-step B — Table Mapping Lookup]
Issue    : [plain-English description of what could not be resolved]
Evidence : [quote the relevant KB text or Excel row that caused the conflict]

Options:
  A) [Most plausible resolution, with rationale]
  B) [Alternative resolution, with rationale]
  C) Mark this attribute [UNRESOLVED] in the output and continue
  D) I'll provide more context (paste corrected mapping row or KB excerpt)

Reply with A, B, C, or D.
```

**Rules:**
- One ambiguity per prompt. Never batch multiple attributes in one question.
- If C is chosen: write `[UNRESOLVED — <summary>]` in `notes`, set `verified_logic = "No"`, continue.
- If D is chosen: re-read the provided context and reattempt before re-asking.
- Do not advance to the next attribute until the current one is resolved or tagged.

---

## Validation Logic

Run these checks **after processing all attributes** and **before writing the Excel file**.
If any check fails, report it and do not write the file until resolved.

### Row-Level Checks (per attribute)
- [ ] `field_name` matches exactly a row in `gold_attributes_excel` — no invented names.
- [ ] `sas_source_table` appears verbatim in at least one KB `.md` file (or is flagged `[NOT FOUND IN KB]`).
- [ ] `databricks_table_name` appears verbatim in `table_mapping_excel` (or is flagged).
- [ ] `databricks_column_name` appears verbatim in `column_mapping_excel` scoped to `databricks_table_name` (or is flagged).
- [ ] `verified_logic = "Yes"` only when `sas_source_table`, `databricks_table_name`, and `databricks_column_name` are all confirmed clean.
- [ ] Multi-source attributes: all source columns are listed comma-separated; none omitted.
- [ ] Any flag in columns 7–11 has a corresponding entry in `notes`.

### File-Level Checks
- [ ] Row count in output = row count in `gold_attributes_excel` — no attributes dropped or duplicated.
- [ ] `verified_logic` column contains only `"Yes"` or `"No"` — no blanks.
- [ ] `transformation` column is non-empty for every attribute where `verified_logic = "Yes"`.
- [ ] No cell contains invented text not traceable to KB or mapping files.
- [ ] `[UNRESOLVED]` count in output matches the count of Ambiguity Protocol triggers where option C was chosen.

---

## Self-Improvement Loop

After writing the output file, perform the following.

### 1. Session Debrief
Count and categorize all `verified_logic = "No"` rows by failure type:
- Not found in KB
- Found in KB but not in table map
- Found in table map but not in column map
- Ambiguous lineage (multiple matches)
- Macro-driven dynamic naming

### 2. Generate Improvement Proposal
Write `gold_attribute_mapping_proposal_v[N].md` alongside the output file:

```markdown
# Attribute Mapper — Improvement Proposal v[N]
Generated after: [run date / gold table name]

## Mapping Failure Summary
| Failure Type | Count | Attributes Affected |
|---|---|---|

## Missing Table Mappings
(Table names found in KB but absent from table_mapping_excel)

## Missing Column Mappings
(Column names found in KB but absent from column_mapping_excel)

## KB Gaps
(Gold attributes not found in any knowledge base MD)

## Suggested Rule Additions
| Issue Observed | Proposed Guardrail or Step |
|---|---|
```

### 3. Present to User

```
Mapping Complete
─────────────────────────────────────────────────────
Total attributes  : N
verified = Yes    : N
verified = No     : N
Improvement proposal saved to: gold_attribute_mapping_proposal_v[N].md

What would you like to do?
  A) Review verified=No rows now and attempt manual resolution
  B) Accept the output as-is and close
  C) Update the mapping Excel files and re-run affected attributes only
```

---

## Quality Checklist

Before presenting the output file, confirm every item:

- [ ] All 5 inputs collected and confirmed before processing began.
- [ ] Pre-flight summary was printed and reviewed with the user.
- [ ] Every gold attribute was processed — none skipped.
- [ ] No Databricks or SAS names invented — all confirmed from source files.
- [ ] Multi-source attributes have all source columns recorded.
- [ ] `verified_logic` column contains only `"Yes"` or `"No"`.
- [ ] Every `"No"` row has a populated `notes` cell explaining why.
- [ ] `transformation` column expresses logic in plain English AND as Databricks-equivalent SQL using confirmed names only.
- [ ] Row count in output matches row count in `gold_attributes_excel`.
- [ ] All `[UNRESOLVED]` items are logged in the improvement proposal.
- [ ] Output file saved to `/mnt/user-data/outputs/gold_attribute_mapping.xlsx`.
- [ ] Improvement proposal generated and path reported to user.
