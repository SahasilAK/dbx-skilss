---
name: sas-script-analyzer
description: >
  Use this skill whenever the user provides a folder of SAS scripts and asks to analyze,
  document, audit, or prepare them for migration or mapping. Triggers include: "analyze
  my SAS folder", "document these SAS scripts", "build a knowledge base from SAS",
  "trace lineage in SAS", "prepare SAS for mapping", "create MD files from SAS scripts",
  "understand what these SAS scripts do", or any request involving .sas files where the
  goal is structured documentation, lineage tracing, or downstream mapping prep.
  ALWAYS trigger this skill — do not attempt to parse SAS scripts without it.
---

# SAS Script Analyzer — Operational Instructions

## Purpose

This skill reads SAS scripts from a user-provided folder one by one and produces
structured Markdown knowledge base files per script. These MD files are the authoritative
documentation artifact — downstream agents consume them instead of raw SAS code.

---

## Step-by-Step Workflow

### Step 1 — List All `.sas` Files

```bash
ls /path/to/sas_folder/*.sas
```

Print the full list to the user, then proceed. If no `.sas` files are found, stop and
report the issue.

### Step 2 — Classify Every Script

For each filename, apply these rules **before opening any file**:

| Contains in name | Classification | Action |
|---|---|---|
| `_ext_` | **EXT** | Do NOT open. Infer source table from name only. |
| `_trn_` | **TRN** | Open and parse fully. |
| Neither | **OTHER** | Open and parse; treat like TRN. |

**Script name = Final output table name** for that script.

**EXT source table inference rule (no file read needed):**
1. Split the script name on `_`.
2. Take the last token. If it is a single letter, use the second-to-last token instead.
3. That token is the inferred source table name.

> Example: `aa_ext_bb_cc_customers` → source table = `customers`
> Example: `aa_ext_bb_orders_v` → source table = `orders` (last token `v` is single letter)

### Step 3 — Parse Each TRN / OTHER Script

Open and fully read each non-EXT script. For each script, extract:
- All input table/view references (see **SAS Parsing Guide** below)
- All output table names
- Every transformation block (PROC SQL or DATA step)
- All conditional logic (WHERE, IF/THEN/ELSE, CASE WHEN)
- Business-relevant macro usage only (see Macro Handling rule)

### Step 4 — Resolve EXT References

When a TRN script references an EXT table, do **not** open that EXT script.
Instead, apply the EXT naming rule from Step 2 to record the inferred source table.

### Step 5 — Trace Lineage Chains

For every output attribute in a TRN script:
1. If the attribute comes from another TRN script, locate that script in the folder and
   trace back through it.
2. Continue until you reach an EXT reference or a hard-coded literal.
3. Record the full chain: `TRN_script_A → TRN_script_B → EXT_table → source_table`.
4. If a referenced TRN script is not found in the folder, flag it as a missing dependency.

### Step 6 — Generate One MD File Per Script

Use the **Output Template** (Section below) to write one file per script.

- File naming: `[script_name]_knowledge.md`
- Save to: `[sas_folder]/knowledge_base/[script_name]_knowledge.md`

Process scripts in dependency order where possible (EXT first, then TRN scripts that
depend only on EXT, then deeper TRN chains).

### Step 7 — Report Missing Dependencies

After all files are written, print a summary:
```
Knowledge Base Generation Complete
-----------------------------------
Scripts processed : N
MD files written  : N
Missing dependencies:
  - script_foo references trn_bar (not found in folder)
  - script_baz references trn_qux (not found in folder)
```

---

## Script Classification Rules (Exact)

**EXT scripts** (`_ext_` in name):
- Extraction only — they SELECT columns from a source table.
- Do NOT trace their internal logic. Infer source table from name (see Step 2).

**TRN scripts** (`_trn_` in name):
- Transformation scripts. Trace all logic within them.
- If a TRN references another TRN not in the folder → flag as missing dependency.

**OTHER scripts** (neither `_ext_` nor `_trn_`):
- Parse fully; treat as TRN for documentation purposes. Mark type as OTHER.

**Macro Handling Rule:**
- Ignore utility macros (logging, timing, environment setup, error handling).
- Document a macro **only if** it directly affects: output column values, filter
  conditions, derived fields, or table name construction.

---

## SAS Parsing Guide

### Identifying Input Tables

| SAS Construct | How to Find Input |
|---|---|
| `DATA step` | `SET libname.tablename;` or `MERGE libname.a libname.b;` |
| `PROC SQL` | `FROM tablename` or `JOIN tablename` clauses |
| `PROC SQL VIEW` | `CREATE VIEW v AS SELECT ... FROM tablename` |
| `%LET` + macro ref | `%LET src = table; FROM &src.` — resolve macro var |
| `LIBNAME` | Defines a library alias; `libname.table` → library prefix + table name |

Extract the table names (strip library prefixes for the source column in the MD table;
note the library in the Notes column).

### Identifying Output Tables

| Construct | Output Table |
|---|---|
| `DATA output_table;` | `output_table` |
| `CREATE TABLE libname.output AS` | `output` |
| `INTO :macro_var` | Not a table — it's a macro variable, document separately |

### Reading PROC SQL Blocks

1. Read `SELECT` clause → list all output columns with any aliases.
2. Read `FROM` / `JOIN` clauses → list all input tables and join types.
3. Read `WHERE` clause → document as a filter condition.
4. Read `GROUP BY` / `HAVING` → document as aggregation logic.
5. For each `CASE WHEN ... END AS col_alias`, record: column name, conditions, values.
6. For subqueries, treat the inner query as a nested input — document it in Section 4.

### Reading DATA Step Logic

1. `SET` / `MERGE` → input sources.
2. Assignment statements (`col = expression`) → derived columns.
3. `IF condition THEN col = val; ELSE col = val2;` → conditional derivation.
4. `RETAIN col;` → signals that `col` carries over across rows — note this explicitly.
5. `ARRAY arr{n} col1–coln;` → document the array and which columns it covers.
6. `OUTPUT;` inside an IF block → conditional row inclusion in output.

### Tracing Derived Columns Through Aliases

When a column appears with an alias in one step and is used by name in a later step:
- Map alias → original expression.
- In Section 3 (Attribute Inventory), record both the final output column name AND the
  original source column name, with the intermediate alias noted in the Transformation
  Logic column.

---

## Output Template

For each script, generate exactly this structure:

````markdown
# [script_name].md — Knowledge Base Entry

## 1. Script Overview
- **Script Name:** [script_name]
- **Output Table Name:** [script_name]  ← same as script name
- **Script Type:** [EXT | TRN | OTHER]
- **Purpose:** (1–2 sentence plain-English summary of what this script produces)

## 2. Input Tables / Sources
| Table/View Name | Type (EXT/TRN/Source) | Inferred Source Table (if EXT) | Notes |
|---|---|---|---|
| ...             | ...                    | ...                            | ...   |

## 3. Output Table: Attribute Inventory
| Output Column | Data Type (if known) | Source Table | Source Column | Transformation Logic | Conditions Applied |
|---|---|---|---|---|---|
| ...           | ...                  | ...          | ...           | ...                  | ...                |

## 4. Transformation Logic Blocks
(One paragraph per major DATA step or PROC SQL block. Reference column names.
Cover: joins and join keys, filters applied, aggregations, derived fields, CASE logic.)

## 5. Conditional Logic Summary
(Bullet list of every IF/THEN/ELSE, WHERE clause, CASE WHEN block with business meaning.
Format: `IF [condition] → [result]`)

## 6. Cross-Script Lineage
| Attribute | Derived From Script | Original Source Table | Lineage Path |
|---|---|---|---|
| ...       | ...                 | ...                   | TRN_a → EXT_b → source_table |

(Only needed for attributes whose source is another TRN script.)

## 7. Macro Usage (Business-Relevant Only)
| Macro Name | Purpose | Affects Attributes |
|---|---|---|
| ...        | ...     | ...                |

(Omit this section entirely if no business-relevant macros are used.)

## 8. Missing Dependencies
- `script_name` — referenced as input but not found in folder.
(Omit if none.)

## 9. Agent Notes
(Any context an agent needs to build a mapping sheet: quirks, implicit assumptions,
multi-step derivations, dynamic table naming, columns with ambiguous lineage.)
````

---

## Edge Case Handling

| Situation | How to Handle |
|---|---|
| Script references a table not in the folder | Note in Section 8 (Missing Dependencies). In Section 2, mark type as "Unknown — not in folder". |
| Macro generates a table name dynamically | Document macro in Section 7, mark output table as `[dynamic — see macro %MACRO_NAME]` in Section 1. |
| Same attribute transformed across multiple DATA steps | Document each step in Section 4; record the full chain in Section 6. |
| EXT script opened accidentally | Close it. Apply naming rule only. Do not document its contents. |
| Column renamed multiple times via aliases | Track all aliases in Section 3 Transformation Logic column; show the full rename chain. |
| `%INCLUDE` references another SAS file | Treat the included file as an inline code block for parsing purposes. Note the include path in Section 9. |
| `PROC SQL VIEW` (not a table) | Treat the view name as the output table name. Note "VIEW" in Data Type column for all attributes. |

---

## Worked Example

### EXT Script: `aa_ext_cc_dd_accounts`

No file read. Apply naming rule:
- Tokens: `aa`, `ext`, `cc`, `dd`, `accounts`
- Last token: `accounts` (not a single letter)
- **Inferred source table:** `accounts`

In any TRN script that references this EXT, Section 2 entry looks like:

| Table/View Name | Type | Inferred Source Table | Notes |
|---|---|---|---|
| aa_ext_cc_dd_accounts | EXT | accounts | Extraction only; no transform |

---

### TRN Script Snippet: `bb_trn_customer_summary`

```sas
PROC SQL;
  CREATE TABLE work.bb_trn_customer_summary AS
  SELECT
    a.customer_id,
    a.region,
    SUM(b.order_amount) AS total_spend,
    CASE WHEN SUM(b.order_amount) > 10000 THEN 'HIGH'
         WHEN SUM(b.order_amount) > 5000  THEN 'MED'
         ELSE 'LOW' END AS spend_tier
  FROM aa_ext_cc_dd_accounts AS a
  JOIN cc_trn_orders AS b
    ON a.customer_id = b.customer_id
  WHERE a.region IN ('NORTH','SOUTH')
  GROUP BY a.customer_id, a.region;
QUIT;
```

**Section 3 output (partial):**

| Output Column | Data Type | Source Table | Source Column | Transformation Logic | Conditions Applied |
|---|---|---|---|---|---|
| customer_id | — | aa_ext_cc_dd_accounts | customer_id | Direct pass-through | region IN ('NORTH','SOUTH') |
| region | — | aa_ext_cc_dd_accounts | region | Direct pass-through | region IN ('NORTH','SOUTH') |
| total_spend | NUM | cc_trn_orders | order_amount | SUM(order_amount) aggregated per customer | WHERE + GROUP BY |
| spend_tier | CHAR | cc_trn_orders | order_amount | CASE WHEN on SUM(order_amount): >10000→HIGH, >5000→MED, else LOW | Derived from total_spend |

**Section 5 entry:**
- `WHERE a.region IN ('NORTH','SOUTH')` → Filters to customers in NORTH or SOUTH regions only.
- `CASE WHEN SUM(order_amount) > 10000 THEN 'HIGH' ...` → Classifies customers by total spend tier.

**Section 6 entry** (assuming `cc_trn_orders` traces to `aa_ext_cc_ee_orders` → source `orders`):

| Attribute | Derived From Script | Original Source Table | Lineage Path |
|---|---|---|---|
| total_spend | cc_trn_orders | orders | bb_trn_customer_summary → cc_trn_orders → aa_ext_cc_ee_orders → orders |
| spend_tier | cc_trn_orders | orders | bb_trn_customer_summary → cc_trn_orders → aa_ext_cc_ee_orders → orders |

---

## Quality Checklist

Before writing each `_knowledge.md` file, verify every item:

- [ ] **Section 1** — Output table name matches script filename exactly.
- [ ] **Section 2** — Every table referenced in FROM, SET, or MERGE is listed. Library prefixes are stripped from table name; noted in the Notes column.
- [ ] **Section 3** — Every SELECT column or DATA step output variable is listed. No column is left undocumented.
- [ ] **Section 3** — Transformation logic column is not empty for derived fields (CASE, SUM, calculated expressions).
- [ ] **Section 4** — Each major PROC SQL or DATA step block has a plain-English paragraph. No raw SAS code pasted here.
- [ ] **Section 5** — Every WHERE clause, IF/THEN/ELSE, and CASE WHEN is captured with its business meaning stated.
- [ ] **Section 6** — If any input is from a TRN script, the full lineage chain is traced to an EXT reference or literal source.
- [ ] **Section 7** — Only macros that affect output values, filters, or table names are documented. Utility macros are omitted.
- [ ] **Section 8** — Every referenced table/script not found in the folder is listed.
- [ ] **EXT scripts** — Source table was inferred from naming rule only. File was not opened.
- [ ] **Macros with dynamic table names** — Documented in Section 7 and flagged in Section 1.
- [ ] **File saved** to `knowledge_base/[script_name]_knowledge.md`.
