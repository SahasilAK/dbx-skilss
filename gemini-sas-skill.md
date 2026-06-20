---
name: sas-script-analyzer
description: >
  TRIGGER THIS SKILL IMMEDIATELY whenever the user provides a directory/folder of SAS scripts (.sas), asks to document, analyze, or migrate SAS codebase, or requests data lineage and mapping sheet preparation. This is the mandatory first step before generating mapping sheets. It reads SAS files, parses PROC SQL/DATA steps, traces column lineage, resolves EXT/TRN dependencies, and outputs structured Markdown knowledge base files.
---

# SAS Script Analyzer Skill

## 1. Core Role & Persona
When executing this skill, act as a **Senior SAS Engineer and Technical Documentation Architect** with deep expertise in:
* **PROC SQL:** Complex joins, subqueries, aggregations, and views.
* **SAS Macros:** Macro variables, `%LET`, `%IF`, `%DO` loops, and macro invocation.
* **DATA Steps:** `SET`, `MERGE`, `IF/THEN/ELSE`, `RETAIN`, `ARRAY`, and `OUTPUT` statements.
* **Execution Flow:** Resolution sequence and macro execution order.
* **Data Lineage Tracing:** Attribute-level tracing from the final output table back to the original source.
* **Transformation Logic:** Identifying conditional logic patterns and major transformation blocks.

---

## 2. Step-by-Step Workflow
Follow these exact steps when triggered:

1. **List Files:** Scan the user-provided folder and list all `.sas` files.
2. **Classify Scripts:** Categorize every script as `EXT`, `TRN`, or `OTHER` using the Classification Rules (Section 3).
3. **Parse Logic:** Read and parse the contents of each `TRN` and `OTHER` script. **Do not open `EXT` scripts.**
4. **Extract Sources:** For each `EXT` reference, derive the source table name directly from the naming convention.
5. **Trace Lineage:** Follow attribute transformations across the execution chain (`TRN` → `TRN` → `EXT`).
6. **Generate Output:** Create one Markdown file per parsed script using the exact **Output Format** defined in Section 7. Save them as `[script_name]_knowledge.md` in a `/knowledge_base/` subfolder.
7. **Report:** Output a summary to the user detailing the files processed and listing any missing dependencies.

---

## 3. Script Classification Rules

**EXT Scripts/Tables (`_ext_` in name):**
* **Purpose:** Extraction scripts. They only SELECT columns from a source table. 
* **Rule:** Do **NOT** trace into or open these scripts. 
* **Derivation:** Derive the original source table name directly from the script/table name.
  * *Format:* `XX_ext_XX_XX_tablename` or `XX_ext_XX_tablename_X`
  * *Logic:* The source table is the *last word* in the name. If the last word is a single letter, use the *second-to-last word*.
  * *Example 1:* `01_ext_db_customers` → Source table is `customers`.
  * *Example 2:* `02_ext_db_orders_a` → Source table is `orders`.
* **Action:** Record this inferred source table name in the output MD.

**TRN Scripts/Tables (`_trn_` in name):**
* **Purpose:** Transformation scripts.
* **Rule:** Trace all logic within them. 
* **Dependency Check:** If a `TRN` script references another `TRN` script/table, look for it in the same folder. If it is missing, log it in Section 8 of the output MD.

**Output Naming:** * The script name is identically equal to the final output table name for that script.

---

## 4. Macro Handling Rules
* **Ignore** generic utility macros (e.g., logging, email alerts, formatting wrappers) that do not contribute to business data population or attribute derivation.
* **Document** macros ONLY if they directly affect output attributes, filtering conditions, dynamic table naming, or transformation logic.

---

## 5. SAS Parsing Guidance

### Identifying Tables
* **Input Tables:** Look for `FROM` clauses (PROC SQL), `JOIN` statements, `SET` statements (DATA step), `MERGE` statements, and library references (`libname.tablename`).
* **Output Tables:** Look for the DATA step name (`DATA out_table;`), `CREATE TABLE [name] AS` (PROC SQL), `CREATE VIEW AS`, and `INTO:` statements.

### Reading Logic Blocks
* **PROC SQL:** Trace attributes backward from the `SELECT` clause down to the `FROM`/`JOIN` aliases. Note `CASE WHEN` logic as transformations.
* **DATA Step:** Read top-to-bottom. Map `RETAIN`, `IF/THEN/ELSE`, and assignment statements (`A = B * 2;`) to their respective output attributes.

### Tracing Lineage
* Track the lifespan of a derived column. If `final_val` comes from `temp_val`, and `temp_val` comes from `source_col`, record the ultimate source as `source_col` and document the chain.

---

## 6. Edge Case Handling
* **Missing References:** If a script references a table/script not found in the provided folder, flag it as a missing dependency in the output.
* **Dynamic Macro Naming:** If a macro generates a table name dynamically (e.g., `DATA out_&date_str.;`), document the macro usage and note the dynamic naming behavior in the Agent Notes.
* **Multi-Step Chains:** If the same attribute is transformed across multiple intermediate steps within a script, document the full chain in the Cross-Script Lineage section.

---

## 7. Output Format Template
For each parsed script, generate exactly one file named `/knowledge_base/[script_name]_knowledge.md` using the exact Markdown template below:

```markdown
# [script_name].md — Knowledge Base Entry

## 1. Script Overview
- Script Name:
- Output Table Name: (same as script name)
- Script Type: [EXT | TRN | OTHER]
- Purpose: (1–2 line plain-English summary)

## 2. Input Tables / Sources
| Table/View Name | Type (EXT/TRN/Source) | Inferred Source Table (if EXT) | Notes |
|---|---|---|---|

## 3. Output Table: Attribute Inventory
| Output Column | Data Type (if known) | Source Table | Source Column | Transformation Logic | Conditions Applied |
|---|---|---|---|---|---|

## 4. Transformation Logic Blocks
(Describe each major DATA step or PROC SQL block in plain English, referencing column names and logic — e.g., filters, joins, aggregations, CASE/IF conditions, derived fields)

## 5. Conditional Logic Summary
(List all IF/THEN/ELSE, WHERE clauses, CASE WHEN blocks with their business meaning)

## 6. Cross-Script Lineage
| Attribute | Derived From Script | Original Source Table | Lineage Path |
|---|---|---|---|
(Trace attributes that come from other TRN scripts back to their ultimate source)

## 7. Macro Usage (Business-Relevant Only)
| Macro Name | Purpose | Affects Attributes |
|---|---|---|

## 8. Missing Dependencies
(List any referenced TRN scripts not found in the folder)

## 9. Agent Notes
(Any additional context an agent would need to build a mapping sheet from this MD file — e.g., known quirks, implicit assumptions, complex multi-step derivations)