---
name: sas_engineer
description: >
  Use this skill whenever a SAS script or SAS program is involved in any way. Triggers:
  analyzing, reading, reverse-engineering, documenting, or explaining a SAS file; tracing where
  a variable or column comes from; explaining PROC SQL or DATA step logic; mapping data lineage
  across SAS scripts; building a knowledge base from a set of SAS programs; generating mapping
  sheets or impact analysis from SAS outputs; auditing macros or business rules embedded in SAS
  code; converting SAS logic to another language. Use this skill for fragments and partial scripts
  too — partial analysis is still valuable. For systems with multiple SAS scripts, apply the
  knowledge base protocol in Section 12 from the start.
---

# SAS Engineer — Analysis & Knowledge Base Reference

## How to Use This Skill

This skill has two modes:

**Single-script mode** (Sections 1–11): Use when analyzing one SAS script in isolation, answering
a targeted question, or producing a quick explanation. Work through sections in order or jump
directly to the relevant section.

**Knowledge base mode** (Section 12): Use when the system contains multiple SAS scripts (5+).
Analyze each script once, write structured `.md` artifacts, then use those artifacts for all
downstream tasks. Never re-read raw `.sas` files after Phase 1 is complete.

**Self-refinement** (Section 13): After every analysis session, update the skill's learned
patterns log. This is mandatory — the skill improves through use.

---

## Table of Contents

1. [Program Structure — Big Picture Orientation](#1-program-structure--big-picture-orientation)
2. [Step Classification](#2-step-classification)
3. [DATA Step Analysis](#3-data-step-analysis)
4. [PROC SQL Analysis](#4-proc-sql-analysis)
5. [Other PROC Steps](#5-other-proc-steps)
6. [Macro Analysis](#6-macro-analysis)
7. [Attribute and Variable Analysis](#7-attribute-and-variable-analysis)
8. [Lineage Backtracking](#8-lineage-backtracking)
9. [Dataset Flow Mapping](#9-dataset-flow-mapping)
10. [Patterns and Anti-Patterns](#10-patterns-and-anti-patterns)
11. [Output Formats](#11-output-formats)
12. [Knowledge Base Mode — Multi-Script Systems](#12-knowledge-base-mode--multi-script-systems)
13. [Self-Refinement Protocol](#13-self-refinement-protocol)
14. [Validation Checks](#14-validation-checks)
15. [Doubt Escalation Protocol](#15-doubt-escalation-protocol)

---

## 1. Program Structure — Big Picture Orientation

**Always do this first, before analyzing any individual block.**

### 1.1 Full-Script Scan

Read the entire script top-to-bottom without deep analysis. Build answers to:
- What is the overall shape? Linear pipeline? Macro-driven loop? Conditional branches?
- How many steps are there?
- What are the terminal inputs (external sources) and terminal outputs (final datasets/files)?
- Are there `%INCLUDE` statements pulling in external files? Note their paths — they must be
  analyzed separately.

### 1.2 Extract Global Declarations

These define the runtime environment. Extract them before analyzing any step.

| Declaration | What to Extract |
|---|---|
| `LIBNAME libref 'path'` | Maps a shorthand (libref) to a physical path or database. Every `libref.dataset` reference resolves here. Note the engine if specified (Oracle, DB2, Teradata, ODBC). |
| `FILENAME fileref 'path'` | Maps a fileref to a flat file, pipe, FTP endpoint, or catalog. |
| `%LET var = value;` | Macro variable set at global scope. Substituted as text everywhere `&var` appears. |
| `%GLOBAL var;` | Declares a variable as global — may be populated later by `CALL SYMPUTX`. |
| `OPTIONS ...;` | Runtime flags. Key ones: `COMPRESS=` (storage), `OBS=` (row limit for testing), `MPRINT`/`MLOGIC`/`SYMBOLGEN` (macro debugging). |
| `%INCLUDE 'file.sas';` | Inlines an external SAS file. Flag it — that file needs its own analysis. |

### 1.3 Build the Step Inventory

Number every step sequentially. This is the skeleton all subsequent analysis hangs from.

```
Step 01  LIBNAME       — Maps SRC, OUT libraries
Step 02  %LET          — Sets &report_year = 2024, &adj_factor = 1.15
Step 03  DATA          — Reads SRC.raw_txn, filters active records → WORK.filtered_txn
Step 04  PROC SORT     — Sorts WORK.filtered_txn by customer_id, date DESC → WORK.sorted_txn
Step 05  PROC SQL      — Joins with SRC.customers, derives amount_adj → WORK.joined
Step 06  PROC MEANS    — Aggregates revenue by region/product → WORK.summary
Step 07  DATA          — Formats and labels → OUT.final_report
Step 08  PROC EXPORT   — Writes OUT.final_report to D:\reports\output.csv
```

---

## 2. Step Classification

Every block of SAS code is one of these types. Classify correctly before analyzing.

### 2.1 DATA Step
- Starts: `DATA <output_dataset(s)>;` — Ends: `RUN;`
- **Execution model**: processes one observation (row) at a time in a loop. Variables reset to
  missing at the top of each iteration unless `RETAIN`ed. This is the single most important
  concept for understanding DATA step logic.
- Reads from `SET`, `MERGE`, `UPDATE`, or raw `INPUT`.
- Can write to multiple output datasets simultaneously via conditional `OUTPUT` statements.

### 2.2 PROC SQL Step
- Starts: `PROC SQL;` — Ends: `QUIT;`
- May contain multiple SQL statements — each is a sub-unit, analyzed separately.
- Set-based execution (processes all rows at once, unlike DATA step).
- SAS extensions: `CALCULATED`, `MONOTONIC()`, `CONNECTION TO`, `INPUT()`, `PUT()`, `SYMGET()`.

### 2.3 Non-SQL PROC Steps

| PROC | Key Statements | Purpose |
|---|---|---|
| `PROC SORT` | `BY`, `NODUPKEY`, `NODUP`, `DUPOUT=` | Sort and optionally de-duplicate. |
| `PROC MEANS` / `PROC SUMMARY` | `CLASS`, `VAR`, `OUTPUT OUT=` | Aggregate numerics. SUMMARY suppresses printed output by default. |
| `PROC FREQ` | `TABLES` | Frequency and cross-tabulation. |
| `PROC TRANSPOSE` | `BY`, `ID`, `VAR`, `PREFIX=` | Pivot wide↔long. |
| `PROC REPORT` | `COLUMN`, `DEFINE`, `COMPUTE` | Reporting; `COMPUTE` blocks can derive new columns. |
| `PROC FORMAT` | `VALUE`, `INVALUE`, `PICTURE` | Creates value-mapping catalogs. |
| `PROC IMPORT` / `PROC EXPORT` | `DATAFILE=`, `DBMS=`, `OUT=` | Read/write external files (CSV, Excel, etc.). |
| `PROC DATASETS` | `MODIFY`, `RENAME`, `DELETE`, `APPEND` | Dataset metadata management without reading data. |
| `PROC APPEND` | `BASE=`, `DATA=` | Appends DATA= rows to BASE= dataset. |
| `PROC COPY` | `IN=`, `OUT=` | Copies datasets between libraries. |
| `PROC TABULATE` | `CLASS`, `VAR`, `TABLE` | Multi-dimensional summary tables. |
| `PROC REG` / `PROC LOGISTIC` / statistical PROCs | Model statements | Statistical modeling — note output datasets (`OUTEST=`, `OUTPUT OUT=`). |

### 2.4 Macro Definitions and Calls
- Definition: `%MACRO name(param1=, param2=); ... %MEND name;`
- Call: `%name(param1=value, param2=value);`
- Macros are text substitution — they generate SAS code before execution. Mental model: unroll
  the macro into the SAS code it produces, then analyze that code.
- Macro variables (`&name`) resolve at compile time, before any data is read.

### 2.5 Macro Flow Control
```sas
%IF &condition %THEN %DO;
  /* Code block A — only generated if condition is true */
%END;
%ELSE %DO;
  /* Code block B */
%END;
```
Always determine which branch is active before analyzing the contents. An inactive branch
produces no SAS code and has no effect on data.

---

## 3. DATA Step Analysis

### 3.1 Input Sources

**SET statement:**
```sas
DATA output;
  SET lib.source (WHERE=(status='ACTIVE') KEEP=id name amount RENAME=(name=cust_name));
RUN;
```
Extract in order:
- Source dataset (`lib.source` → resolve libref).
- `WHERE=`: filter applied at read time. Only matching rows enter the PDV (Program Data Vector).
- `KEEP=` / `DROP=`: column projection at read time. Missing columns never enter the PDV.
- `RENAME=`: applied at read time. The new name is what the DATA step body sees.
- `FIRSTOBS=` / `OBS=`: row range — often used for testing, flag if present in production code.

**MERGE statement:**
```sas
DATA merged;
  MERGE left_ds(IN=a) right_ds(IN=b);
  BY key1 key2;
  IF a AND b;
RUN;
```
- `IN=` variables are Boolean flags: `1` if the current row came from that dataset.
- Both datasets **must be pre-sorted** by the `BY` variables (or indexed). If no preceding
  PROC SORT is visible, flag as ⚠️ MISSING PRE-SORT.
- Join type from `IF` condition:

| IF Condition | Join Type |
|---|---|
| `IF a;` | Left join — all left rows kept |
| `IF b;` | Right join — all right rows kept |
| `IF a AND b;` | Inner join — only matching rows |
| `IF a OR b;` | Full outer join |
| `IF NOT b;` | Left anti-join — left rows with no match |
| `IF NOT a AND b;` | Right anti-join |
| *(no IF)* | Full outer join (SAS default MERGE behavior) |

**UPDATE statement:**
```sas
DATA master;
  UPDATE master_ds transaction_ds;
  BY key;
RUN;
```
Applies changes from `transaction_ds` onto `master_ds`. Missing values in `transaction_ds`
do NOT overwrite values in `master_ds` (unlike MERGE).

**INPUT statement (raw data):**
```sas
DATA raw;
  INFILE RAWFILE DLM=',' DSD MISSOVER FIRSTOBS=2;
  INPUT id $ name $ amount;
RUN;
```
Note: `DSD` handles quoted fields; `MISSOVER` prevents reading the next line for missing trailing values;
`FIRSTOBS=2` skips a header row.

### 3.2 The Program Data Vector (PDV)

The PDV is the in-memory row buffer. Understanding it explains all DATA step behavior:

1. At step start, SAS initializes the PDV — all variables set to missing.
2. For each observation, SAS reads input into the PDV.
3. Statements execute top-to-bottom, modifying PDV values.
4. At `RUN;` (or explicit `OUTPUT;`), the PDV is written to the output dataset.
5. The PDV resets to missing and the cycle repeats for the next observation.

**RETAIN breaks the reset** — retained variables keep their value from the previous observation.

### 3.3 Variable Derivation Patterns

| Pattern | Example | Notes |
|---|---|---|
| Arithmetic | `profit = revenue - cost;` | Standard operators: + - * / ** |
| String functions | `clean_id = STRIP(UPCASE(raw_id));` | STRIP removes leading/trailing blanks; COMPRESS removes all specified chars |
| Substring | `prefix = SUBSTR(code, 1, 3);` | 1-based indexing in SAS |
| Concatenation | `full_name = CATX(' ', first, last);` | CATX inserts delimiter, skips missing |
| Scan | `word2 = SCAN(sentence, 2, ' ');` | Extracts Nth word by delimiter |
| Date arithmetic | `age_days = TODAY() - birth_date;` | SAS dates are numeric (days since 1Jan1960) |
| Date conversion | `dt = INPUT('01JAN2024', DATE9.);` | Character → SAS date numeric |
| Date formatting | `dt_str = PUT(date_val, DATE9.);` | SAS date numeric → character |
| Date parts | `yr = YEAR(date); mo = MONTH(date);` | Extracts components |
| Date shifting | `next_mo = INTNX('MONTH', date, 1, 'B');` | 'B'=beginning, 'E'=end, 'S'=same day |
| Interval count | `months = INTCK('MONTH', start, end);` | Count of interval boundaries crossed |
| Conditional | `IF x > 0 THEN flag = 1; ELSE flag = 0;` | Two-branch; use SELECT for multi-branch |
| Inline conditional | `flag = IFN(x > 0, 1, 0);` | IFN=numeric result; IFC=character result |
| SELECT/WHEN | See below | Multi-way conditional; equivalent to CASE |
| Running total | `RETAIN cumsum 0; cumsum + amount;` | `var + expr` is shorthand for `var = var + expr` with RETAIN |
| Array loop | See below | Process multiple similarly-named variables |

**SELECT/WHEN block:**
```sas
SELECT (status_code);
  WHEN ('A')       status_label = 'Active';
  WHEN ('I', 'S')  status_label = 'Inactive';
  WHEN (' ')       status_label = 'Missing';
  OTHERWISE        status_label = 'Unknown';
END;
```

**Array processing:**
```sas
ARRAY scores{5} score1-score5;
DO i = 1 TO 5;
  IF scores{i} = . THEN scores{i} = 0;  /* Replace missing with 0 */
END;
DROP i;
```
Arrays are compile-time aliases — they don't create new variables, they provide indexed access
to existing ones. Always `DROP` the loop counter unless needed in output.

### 3.4 Filtering and Row Control

| Mechanism | Behavior |
|---|---|
| `WHERE` statement | Filters rows before they enter the PDV. More efficient than subsetting IF. Cannot reference variables created in the current DATA step. |
| Subsetting `IF` (no THEN) | `IF condition;` — if false, skips to top of loop (row is dropped). Can reference derived variables. |
| `DELETE;` statement | Immediately terminates current iteration; row is dropped. |
| `OUTPUT;` statement | Writes current PDV to output dataset. If used anywhere explicitly, only rows reaching `OUTPUT` are written (implicit output at end of loop is suppressed). |
| `STOP;` statement | Stops the DATA step loop entirely. Used in controlled raw-file reads. |

### 3.5 BY-Group Processing

When `BY var;` appears in a DATA step (without MERGE):
- Automatic variables `FIRST.var` and `LAST.var` are created.
- `FIRST.var = 1` on the first row of each group; `LAST.var = 1` on the last.
- Dataset must already be sorted by those variables.
- Typical patterns: reset accumulators on `FIRST.`, emit rows on `LAST.`.

```sas
/* Emit one summary row per customer */
DATA summary;
  SET detail;
  BY customer_id;
  RETAIN total 0;
  IF FIRST.customer_id THEN total = 0;
  total + amount;
  IF LAST.customer_id THEN OUTPUT;
RUN;
```

### 3.6 Output Dataset Options

On the `DATA` statement itself:
- `KEEP=` / `DROP=`: restrict output columns.
- `RENAME=`: rename columns in the output.
- `LABEL`: `LABEL var = 'Human readable description';`
- `FORMAT`: `FORMAT date_var DATE9.; FORMAT amount COMMA12.2;`
- `LENGTH`: `LENGTH char_var $50;` — always declare before first assignment to avoid truncation.
- `ATTRIB`: combines LENGTH, FORMAT, INFORMAT, LABEL in one statement.

---

## 4. PROC SQL Analysis

### 4.1 Statement Types Inside PROC SQL

A single `PROC SQL; ... QUIT;` block may contain multiple statements. Analyze each independently.

| Statement | Notes |
|---|---|
| `CREATE TABLE lib.out AS SELECT ...` | Main analysis target — creates a SAS dataset. |
| `CREATE VIEW lib.v AS SELECT ...` | Creates a stored query (re-executed on every reference — flag this). |
| `SELECT ...` (no CREATE) | Sends output to the Results Viewer only — no dataset created. |
| `INSERT INTO lib.t VALUES (...);` | Row-level insert. |
| `UPDATE lib.t SET col=val WHERE ...;` | Row-level update. |
| `DELETE FROM lib.t WHERE ...;` | Row-level delete. |
| `DROP TABLE lib.t;` | Drops a dataset. |
| `CONNECT TO dbname (...); ... DISCONNECT FROM dbname;` | Pass-through to external database. |

### 4.2 SELECT Statement — Layer-by-Layer Decomposition

Decompose every SELECT into these layers in order:

#### Layer 1 — SELECT: Output Columns

For each column in the SELECT list, classify it:

| Column Type | How to Identify | What to Document |
|---|---|---|
| Direct pass-through | `alias.column_name` | Source table alias, original column name |
| Renamed | `alias.col AS new_name` | Source column, new name |
| Arithmetic | `a.price * a.qty AS revenue` | Full expression, all input columns |
| CASE/WHEN | `CASE WHEN ... THEN ... END AS label` | All conditions and result values |
| Aggregate | `SUM(a.amount) AS total` | Function, input column, implied GROUP BY |
| CALCULATED | `CALCULATED prev_col * 0.05 AS fee` | Which earlier SELECT alias it references |
| Scalar subquery | `(SELECT MAX(dt) FROM lib.t) AS max_dt` | Analyze the subquery recursively |
| Constant | `'ACTIVE' AS status` or `1 AS flag` | Document as 📌 hard-coded |
| Macro-substituted | `&macro_var AS col` | Trace the macro variable |

**CALCULATED keyword** is SAS-specific. It references a column alias defined earlier in the
same SELECT clause. If `amount_adj` is defined as `amount * 1.15`, then
`CALCULATED amount_adj * 0.05` means `(amount * 1.15) * 0.05`. Always unroll the chain.

#### Layer 2 — FROM: Source Tables

```sql
FROM lib.transactions a
```
- Full dataset reference: `libref.datasetname` — resolve the libref.
- Alias assigned: `a` — all `a.column` references resolve here.
- Dataset options on FROM are valid in SAS: `FROM lib.ds (WHERE= KEEP= RENAME=)`.
- **Subquery in FROM**: treat as a virtual table, analyze the inner SELECT recursively.

#### Layer 3 — JOINs

```sql
LEFT JOIN lib.customers b  ON a.customer_id = b.customer_id
INNER JOIN lib.products c  ON a.product_id = c.product_id AND c.active = 1
```

For each JOIN, document:
- Join type (INNER / LEFT / RIGHT / FULL OUTER / CROSS).
- Joined dataset and its alias.
- Join key(s) — every condition in the ON clause.
- Effect on row count: LEFT keeps all left rows (right columns NULL on no-match); INNER drops
  unmatched rows from both sides; FULL OUTER keeps all rows from both.
- Multi-column join keys: ALL conditions must match simultaneously.
- Non-equi joins (ON a.price BETWEEN b.low AND b.high): flag as range joins — can produce
  fan-out (multiple matching rows per left row).

#### Layer 4 — WHERE: Row Filtering

```sql
WHERE a.date BETWEEN '01JAN2024'D AND '31DEC2024'D
  AND b.region IN ('NORTH', 'SOUTH')
  AND c.status <> 'D'
  AND EXISTS (SELECT 1 FROM lib.approved ap WHERE ap.id = a.id)
```

Document every condition. Flag:
- 📌 Hard-coded date literals (e.g., `'01JAN2024'D`).
- `IN (subquery)` — if the subquery can return NULL, the entire IN clause may silently return
  zero rows (SQL three-valued logic). Flag as ⚠️ NULL IN SUBQUERY RISK.
- `NOT IN (subquery)` — if subquery returns any NULL, NOT IN returns zero rows. High risk.
- `EXISTS` / `NOT EXISTS` — correlated subqueries; analyze the inner SELECT separately.
- Macro variables in WHERE: `WHERE year = &report_year` — document the substituted value.

#### Layer 5 — GROUP BY

```sql
GROUP BY a.region, a.product_type
```
- Lists the grouping keys. Output has one row per unique key combination.
- Every non-aggregate SELECT column must appear here (or be functionally dependent on the keys).
- Defines the **granularity** of the output — document this explicitly.

#### Layer 6 — HAVING

```sql
HAVING SUM(a.amount) > 10000
  AND COUNT(*) >= 5
```
Applied after GROUP BY. Filters on aggregated values. Can reference CALCULATED aliases.

#### Layer 7 — ORDER BY

```sql
ORDER BY a.region ASC, total_revenue DESC
```
Controls row order in the output. Note: SAS datasets do not guarantee preservation of ORDER BY
after subsequent steps unless followed by PROC SORT. Flag if downstream steps assume order.

#### Layer 8 — Set Operations

```sql
SELECT a, b FROM lib.t1
UNION ALL
SELECT a, b FROM lib.t2
```
- `UNION`: de-duplicates. `UNION ALL`: keeps all rows (much faster — prefer this unless
  de-duplication is required).
- `INTERSECT`: rows in both. `EXCEPT`: rows in first but not second.
- Column count and types must match between sets.

### 4.3 SAS SQL Extensions Reference

| Extension | Syntax | Meaning |
|---|---|---|
| CALCULATED | `CALCULATED alias` | References a column alias defined earlier in the same SELECT |
| MONOTONIC() | `MONOTONIC() AS rownum` | Sequential row counter within the query result. Behavior is unreliable in joins — use with caution, flag always |
| Date literal | `'01JAN2024'D` | SAS-specific date literal (numeric value = days since 1Jan1960) |
| INPUT() | `INPUT(str_col, DATE9.)` | Converts character to numeric/date using a SAS informat |
| PUT() | `PUT(date_col, DATE9.)` | Converts numeric/date to character using a SAS format |
| SYMGET() | `SYMGET('macro_var')` | Reads a macro variable value at SQL execution time (runtime, not compile time) |
| COALESCE() | `COALESCE(a, b, c)` | First non-NULL/non-missing value. Standard SQL but critical in SAS for null handling |
| OUTOBS= | `PROC SQL OUTOBS=1000;` | Limits output rows (like LIMIT n or TOP n) |
| NOPRINT | `PROC SQL NOPRINT;` | Suppresses Results Viewer output |
| STIMER | `PROC SQL STIMER;` | Prints per-statement timing — debug flag |
| FEEDBACK | `PROC SQL FEEDBACK;` | Prints expanded SQL after macro substitution — debug flag |

### 4.4 Pass-Through Queries

```sas
PROC SQL;
  CONNECT TO oracle (path='findb' user=&user password=&pw schema=finance);
  CREATE TABLE WORK.gl AS
    SELECT * FROM CONNECTION TO oracle (
      SELECT acct_id, SUM(amount) AS total
      FROM finance.gl_transactions
      WHERE posting_date >= ADD_MONTHS(SYSDATE, -12)
      GROUP BY acct_id
    );
  DISCONNECT FROM oracle;
QUIT;
```

- The inner SQL is **native Oracle SQL** (or DB2, Teradata, SQL Server — match to connection type).
- The source is an external database table — this is a 🔌 system boundary.
- Document: database engine, schema, table name, and any native SQL functions used.
- The result is brought into SAS as `WORK.gl` — analyze it as any other intermediate dataset.

### 4.5 Macro Variables in SQL

Macro variables can appear anywhere in SQL text:

```sql
WHERE year = &report_year          /* numeric — substituted as-is */
  AND status IN (&status_list)     /* may expand to 'A','I','S' — a list */
  AND flag = "&active_flag"        /* character — double quotes allow macro resolution */
```

Single-quoted strings `'&var'` do NOT resolve macro variables. Double-quoted `"&var"` do.
Always trace every `&variable` in SQL to its assignment.

---

## 5. Other PROC Steps

### 5.1 PROC SORT

```sas
PROC SORT DATA=lib.input OUT=lib.output NODUPKEY DUPOUT=lib.dupes;
  BY key1 key2 DESCENDING amount;
RUN;
```

| Element | Notes |
|---|---|
| `DATA=` | Input dataset |
| `OUT=` | Output dataset (omit = sorts in place) |
| `NODUPKEY` | Keeps first row per unique BY-key combination. Which row is "first" depends on the sort order — document this. |
| `NODUP` | Removes rows identical across ALL columns (not just BY keys). Rarely what's intended. |
| `DUPOUT=` | Sends removed duplicate rows here for inspection |
| `DESCENDING` | Applies to the immediately following variable. `BY customer_id DESCENDING date` = ascending customer, descending date within. |

**Lineage note**: PROC SORT is transparent to attribute lineage — no columns are created or
modified. It only changes row order and optionally removes rows (NODUPKEY/NODUP).

### 5.2 PROC MEANS / PROC SUMMARY

```sas
PROC MEANS DATA=lib.detail NWAY NOPRINT;
  CLASS region product_type;
  VAR revenue units cost;
  OUTPUT OUT=lib.summary
    SUM(revenue)=total_rev SUM(units)=total_units
    MEAN(revenue)=avg_rev N(revenue)=row_count
    MAX(cost)=max_cost;
RUN;
```

| Element | Notes |
|---|---|
| `CLASS` | Grouping variables (like GROUP BY). Multiple CLASS variables create all combinations. |
| `VAR` | Numeric variables to aggregate. |
| `NWAY` | Outputs only the most detailed level (all CLASS variables). Without NWAY, SAS also outputs subtotals at each level of the CLASS hierarchy. |
| `OUTPUT OUT=` | Output dataset. Statistic keywords define output column names: `STAT(input_var)=output_name`. |
| `_TYPE_` | Automatic output variable: binary encoding of which CLASS variables are active in that row. |
| `_FREQ_` | Automatic output variable: number of observations in the group. |
| `NOPRINT` | Suppresses the printed output (output dataset still produced). |

Available statistics: `SUM`, `MEAN`, `N`, `NMISS`, `MIN`, `MAX`, `RANGE`, `STD`, `VAR`,
`STDERR`, `CV`, `T`, `PRT`, `USS`, `CSS`, `SKEWNESS`, `KURTOSIS`, `MEDIAN`, `MODE`,
`P1`, `P5`, `P10`, `P25`, `P50`, `P75`, `P90`, `P95`, `P99`, `QRANGE`.

### 5.3 PROC TRANSPOSE

```sas
PROC TRANSPOSE DATA=lib.long OUT=lib.wide PREFIX=month_ NAME=source_col;
  BY customer_id;
  ID period;        /* Values of 'period' become column names: month_JAN, month_FEB, etc. */
  VAR amount;       /* Values of 'amount' fill the new columns */
RUN;
```

| Element | Notes |
|---|---|
| `BY` | Variables that define row identity in the wide output — one row per unique BY combination. |
| `ID` | Variable whose distinct values become the new column names. |
| `VAR` | Variable whose values fill the new columns. Multiple VAR vars = multiple rows per BY group in output. |
| `PREFIX=` | Prepended to ID values. Required when ID values start with digits or contain spaces. |
| `NAME=` | Name for the automatic `_NAME_` variable in output (records which VAR column this row came from). |
| `LABEL=` | Name for the automatic `_LABEL_` variable in output. |

**Lineage note**: The wide columns (`PREFIX_idvalue`) did not exist as columns in the source —
they were row values of the ID variable. Always document this pivot transformation explicitly.

### 5.4 PROC FORMAT

```sas
PROC FORMAT LIBRARY=lib.formats;
  VALUE $status_fmt
    'A'       = 'Active'
    'I', 'S'  = 'Inactive'
    OTHER     = 'Unknown';

  VALUE revenue_tier
    LOW  -<  100  = 'Bronze'
    100  -< 1000  = 'Silver'
    1000 -  HIGH  = 'Gold';

  INVALUE $code_infmt
    'Active'   = 'A'
    'Inactive' = 'I';
QUIT;
```

- Formats are stored in a **format catalog** (`lib.formats` above; default is `WORK.FORMATS`).
- `VALUE` defines display formats (used in `FORMAT` statements and `PUT()` function).
- `INVALUE` defines input formats (used in `INFORMAT` statements and `INPUT()` function).
- `PICTURE` defines numeric picture templates.
- Formats are NOT datasets — they cannot be queried. They are lookup tables for display/conversion.
- When `PUT(var, $status_fmt.)` appears in a DATA step or SQL, the format is performing a value
  translation — document the full mapping as a business rule.

### 5.5 PROC IMPORT / PROC EXPORT

```sas
PROC IMPORT DATAFILE='D:\data\rates.csv' OUT=WORK.rates DBMS=CSV REPLACE;
  DELIMITER=',';
  GETNAMES=YES;
  DATAROW=2;
RUN;
```

- `DATAFILE=`: physical file path — this is a 🔌 external source.
- `DBMS=`: file type (CSV, EXCEL, TAB, DLM, XLSX, ACCESS, JMP).
- `GETNAMES=YES`: uses first row as column names.
- `DATAROW=`: row number where data begins.
- Column types are guessed by SAS — flag character columns that may have been guessed wrong.

### 5.6 PROC DATASETS

```sas
PROC DATASETS LIBRARY=WORK NOLIST;
  MODIFY filtered_txn;
    RENAME old_name = new_name;
    LABEL amount = 'Transaction Amount (USD)';
    FORMAT date DATE9.;
  DELETE temp1 temp2;
QUIT;
```

Modifies dataset metadata without reading data. Changes take effect in the catalog only.
**Lineage note**: `RENAME` inside PROC DATASETS changes the name of a column in a permanent
dataset — this is a lineage break point. The column existed under the old name before this step.

---

## 6. Macro Analysis

### 6.1 Macro Definition Anatomy

```sas
%MACRO process_period(inlib=WORK, outlib=OUT, period=, debug=N);
  %LOCAL i step_count;          /* Local scope — not visible outside macro */
  %IF &debug = Y %THEN %PUT NOTE: Processing period &period;

  /* ... SAS steps ... */

%MEND process_period;
```

Document for every macro:
- Name and all parameters (required vs. optional with defaults).
- `%LOCAL` variables — their scope is limited to this macro invocation.
- `%GLOBAL` variables set inside — these persist after the macro ends.
- What SAS code it generates (unroll loops mentally).
- Which external datasets or macro variables it reads and writes.

### 6.2 Macro Variable Scoping

Resolution order when `&varname` is encountered:
1. Local symbol table of the current macro.
2. Local symbol tables of enclosing macros (nested call stack), innermost first.
3. Global symbol table.

If not found in any scope, SAS issues a WARNING and substitutes the literal `&varname` text —
this often silently corrupts SQL queries or filenames. Always flag unresolved references.

### 6.3 Macro Variable Assignment Methods

| Method | Where Used | Scope | Notes |
|---|---|---|---|
| `%LET var = value;` | Anywhere | Current scope | Simple text assignment |
| `%GLOBAL var;` then `%LET` | Macro body | Global | Explicit global promotion |
| `%LOCAL var;` then `%LET` | Macro body | Local to macro | Prevents polluting global scope |
| `CALL SYMPUT('var', value)` | DATA step | Global (or local if in macro) | Sets from data values. Value available in NEXT step only. |
| `CALL SYMPUTX('var', value)` | DATA step | Global (or local if in macro) | Like SYMPUT but strips leading/trailing spaces. Prefer this. |
| `%SYSFUNC(...)` | Macro expression | Current scope | Executes a DATA step function at macro compile time |

**Critical timing rule**: `CALL SYMPUT` inside a DATA step sets the macro variable AFTER that
DATA step completes. Code in the same DATA step cannot read the new value. Cross-step macro
variable dependencies must be explicitly documented in the knowledge base.

### 6.4 %SYSFUNC Patterns

```sas
%LET today     = %SYSFUNC(TODAY(), DATE9.);
%LET nobs      = %SYSFUNC(ATTRN(%SYSFUNC(OPEN(lib.dataset,I)),NOBS));
%LET file_exists = %SYSFUNC(FILEEXIST('D:\data\input.csv'));
```

- `TODAY()`, `DATE()`, `TIME()`, `DATETIME()` — current timestamps.
- `OPEN()` / `ATTRN()` / `CLOSE()` — dataset attribute introspection.
- `FILEEXIST()` / `FOPEN()` — file system checks.

### 6.5 Iterative Macros

```sas
%MACRO process_all(n=);
  %DO i = 1 %TO &n;
    PROC SQL;
      CREATE TABLE WORK.result_&i AS
        SELECT * FROM lib.source WHERE period = &i;
    QUIT;
  %END;
%MEND;
%process_all(n=12);
```

Mentally unroll: this generates 12 PROC SQL blocks, one for each value of `i`. The output
tables are `WORK.result_1` through `WORK.result_12`. Document all generated datasets.

---

## 7. Attribute and Variable Analysis

### 7.1 Attribute Properties

Every SAS variable (column) has these properties:

| Property | Defined By | Notes |
|---|---|---|
| **Name** | First assignment, `LENGTH`, or `ATTRIB` | Max 32 characters in SAS 9.4+ |
| **Type** | First assignment or `LENGTH` | Numeric or Character — SAS has only two types |
| **Length** | `LENGTH` statement or first assignment | Character: byte count ($1–$32767); Numeric: 3–8 bytes (8 = double precision) |
| **Format** | `FORMAT` statement or `ATTRIB` | Controls how values display (DATE9., COMMA12.2, $STATUS_FMT.) |
| **Informat** | `INFORMAT` statement or `ATTRIB` | Controls how raw text is read in |
| **Label** | `LABEL` statement or `ATTRIB` | Human-readable description, up to 256 characters |

**Length trap**: If a character variable is first assigned a short string (`status = 'Y';`),
its length is set to 1 and longer values will be silently truncated. Always declare `LENGTH`
before the first assignment for character variables.

### 7.2 Attribute Classification

Classify every attribute in the final output before backtracking:

| Class | Definition | Lineage Depth |
|---|---|---|
| **Pass-through** | Read from source, passed unchanged. | One hop — trace to source column. |
| **Renamed pass-through** | Same data, different name. | One hop — find RENAME= to get original name. |
| **Derived — arithmetic** | Computed via math expression. | Trace all operands. |
| **Derived — string** | Result of string function(s). | Trace all function inputs. |
| **Derived — conditional** | IF/ELSE, SELECT/WHEN, CASE/WHEN. | Trace all condition branches and inputs. |
| **Derived — aggregate** | SUM, COUNT, AVG, etc. | Trace input column + grouping keys + filter. |
| **Derived — lookup/format** | PUT(var, format.) or joined from reference table. | Trace source column + format definition or reference table. |
| **Derived — pivot** | Result of PROC TRANSPOSE. | Trace to the VAR column and the ID values. |
| **Macro-injected** | Value substituted from a macro variable. | Trace macro variable to its assignment. |
| **Constant** | Hard-coded literal. | Lineage ends here. Document the value. |
| **System/automatic** | `_N_`, `_ERROR_`, `FIRST.`, `LAST.`, `_TYPE_`, `_FREQ_` | SAS-generated. Document which step they originate from. |

### 7.3 Finding Where an Attribute Is First Defined

Scan the script in this priority order:

1. `ATTRIB var LENGTH=... LABEL=... FORMAT=...;` — full declaration.
2. `LENGTH var $20;` or `LENGTH var 8;` — type and length.
3. `RENAME old = var;` or `RENAME=(old=var)` option — earlier name.
4. First `var = expression;` assignment — type and length inferred from right-hand side.
5. `INPUT` statement — raw data read.
6. Incoming from `SET`/`MERGE` — inherited from source dataset.

---

## 8. Lineage Backtracking

This is the core analytical task. Follow this methodology precisely.

### 8.1 The Backtracking Algorithm

Start from the **target attribute in the final output**. Apply these steps recursively until
every path terminates at an External Source, a Constant, or a System variable.

```
FUNCTION trace(attribute, dataset):

  1. Find the step that writes `dataset`.
  2. Classify `attribute` in that step (Section 7.2).
  3. Based on classification:

     Pass-through:
       → attribute exists in the step's input dataset under the same name
       → RECURSE: trace(attribute, input_dataset)

     Renamed pass-through:
       → find RENAME= option; get original_name
       → RECURSE: trace(original_name, input_dataset)

     Derived — arithmetic/string/conditional:
       → document the full expression
       → for each input column in the expression:
           → RECURSE: trace(input_column, current_step_inputs)

     Derived — aggregate:
       → document function + input column + grouping keys + any WHERE/HAVING filter
       → RECURSE: trace(input_column, current_step_inputs)

     Derived — lookup/format:
       → if PUT(var, fmt.): find PROC FORMAT definition; document the mapping
       → if joined reference table: RECURSE: trace(col, reference_table)
       → RECURSE: trace(source_var_before_lookup, ...)

     Macro-injected:
       → find the %LET / CALL SYMPUTX that sets the macro variable
       → if the value is static: lineage ends (document value as 📌 constant)
       → if the value is computed: RECURSE into the computation

     Constant:
       → STOP. Document: value, where hard-coded, and what it means.

     External Source (LIBNAME points to external path/DB, no producing step in this system):
       → STOP 🔌. Document: system, path/schema, table, column name.
```

### 8.2 Cross-Step Rules

**Crossing a PROC SORT boundary**: transparent — the attribute came from the `DATA=` input
unchanged. Step back to that dataset's producing step.

**Crossing a PROC MEANS boundary**: the output column was derived by applying a statistic to
the `VAR` column over a `CLASS` grouping. The upstream target is the VAR column.

**Crossing a PROC TRANSPOSE boundary**: the output columns (`PREFIX_idvalue`) came from values
of the `VAR` column in the long-format input. The upstream target is the VAR column, filtered
to rows where the `ID` column equals the idvalue.

**Crossing a DATA step MERGE boundary**: determine which input dataset contributed the attribute
by checking the `IN=` flag logic and `BY` keys. If the attribute is only present in one of the
merged datasets, it came from there.

**Crossing a PROC SQL JOIN boundary**: resolve the table alias to its source dataset.
If the column comes from a subquery, descend into the subquery and recurse.

**Crossing a %INCLUDE boundary**: the attribute was defined or transformed in an external file.
Flag as 🔌 EXTERNAL FILE — that file must be analyzed separately.

**Crossing a CALL SYMPUT / CALL SYMPUTX boundary**: a macro variable was set from a data value.
Trace backward to the DATA step expression that produced the value used in CALL SYMPUTX.

### 8.3 RETAIN and Running Value Dependencies

```sas
RETAIN running_total 0;
running_total + amount;
```

`running_total` at observation N is the cumulative sum of `amount` from observations 1 through N.
Its lineage depends on ALL prior rows — not just the current row. Always flag RETAIN-based
attributes explicitly:

```
RETAIN dependency: running_total at row N = SUM(amount, rows 1..N) within current BY group
```

Document: what value initializes the retained variable, what BY-group reset logic exists (if any),
and whether the initial value is a constant, missing, or data-driven.

### 8.4 Lineage Notation Standard

Use this notation consistently in all lineage outputs:

```
FINAL.OUTPUT.ATTRIBUTE_NAME
  ← [Step 07 DATA] Derived-arithmetic: REVENUE * 1.15
      ← REVENUE
          ← [Step 05 PROC SQL] Pass-through: WORK.joined.REVENUE
              ← [Step 03 DATA] Pass-through from source
                  🔌 SRC.raw_txn.REVENUE ← D:\data\source\ (SAS library)
      ← 1.15
          🔧 &adj_factor = 1.15 ← %LET at Step 02 📌 hard-coded
```

**Notation key:**
- `←` : lineage direction (output derived from what's to the right)
- `🔌` : external system boundary
- `📌` : hard-coded constant
- `🔧` : macro variable dependency
- `⚠️` : risk or anomaly
- `[Step N TYPE]` : step reference

### 8.5 Special Lineage Scenarios

**View datasets** (`DATA v / VIEW=v; SET lib.src; RUN;`): a SAS view re-executes its definition
every time it is referenced. It does not store data. Lineage passes through the view definition.

**PROC FORMAT lookups**: `PUT(status, $status_fmt.)` maps a source value through a format.
The lineage is: source column → format mapping → output label. Document the full value map.

**Pass-through SQL results**: rows come from an external database. The lineage terminates at
the external system. Document the full SQL sent to the database (the inner CONNECTION TO query).

**`_N_` (DATA step iteration counter)**: derived attributes involving `_N_` have lineage tied
to the iteration order of the DATA step, which depends on the source dataset's row order.

---

## 9. Dataset Flow Mapping

### 9.1 Flow Entry Format

For each step:
```
[Step 05] PROC SQL
  INPUTS:  WORK.sorted_txn (alias t), SRC.customers (alias c)
  OUTPUTS: WORK.joined
  JOIN:    LEFT JOIN on t.customer_id = c.customer_id
  FILTER:  t.date BETWEEN '01JAN2024'D AND '31DEC2024'D
  DERIVES: amount_adj (amount × 1.15), estimated_fee (amount_adj × 0.05), region_category (CASE)
  GRANULARITY: one row per transaction (txn_id is the key)
```

Always document granularity — it is one of the most important and most frequently omitted facts.

### 9.2 Dataset Classification

| Category | Characteristics |
|---|---|
| **External Source** | Referenced as input; never created by any step in this script. Comes from a physical library, database, or flat file. Mark 🔌. |
| **WORK Intermediate** | Created and consumed within the same script. Temporary (deleted when SAS session ends). |
| **Permanent Intermediate** | Written to a permanent library but also consumed within the same system. |
| **Terminal Output** | Created by this system; not consumed by any step within this system. These are the deliverables. |

### 9.3 Unresolved References

If a step reads from a dataset that is never created by any step in the system being analyzed,
flag it as:
- ⚠️ `UNRESOLVED SOURCE` if you cannot determine where it comes from.
- 🔌 `EXTERNAL SOURCE` if the LIBNAME confirms it comes from outside this system.

---

## 10. Patterns and Anti-Patterns

### 10.1 Patterns to Recognize

**First-occurrence selection (de-dup by latest date):**
```sas
PROC SORT DATA=ds; BY id DESCENDING date; RUN;
DATA deduped;
  SET ds;
  BY id;
  IF FIRST.id;  /* Keeps first = highest date (due to descending sort) */
RUN;
```
The retained record is the most recent by `date`. The sort order determines which record is kept.
Always document: sort key, sort direction, and which record is retained.

**Running totals with group reset:**
```sas
DATA with_running;
  SET detail;
  BY customer_id;
  RETAIN cum_amount 0;
  IF FIRST.customer_id THEN cum_amount = 0;
  cum_amount + amount;
RUN;
```
Reset pattern is correct — `cum_amount` restarts for each customer.

**Macro variable captured from data:**
```sas
/* Step 03: captures max date from data */
PROC SQL NOPRINT;
  SELECT MAX(date) INTO :max_dt FROM WORK.detail;
QUIT;
/* &max_dt is now available to all subsequent steps */
```
`SELECT ... INTO :var` is the PROC SQL equivalent of CALL SYMPUTX. Document the cross-step
dependency: downstream steps that use `&max_dt` depend on Step 03 having run.

**Hash table lookup (DATA step):**
```sas
DATA result;
  IF _N_ = 1 THEN DO;
    DECLARE HASH h(dataset:'lib.lookup');
    h.defineKey('code');
    h.defineData('description');
    h.defineDone();
  END;
  SET lib.main;
  rc = h.find();
  IF rc = 0 THEN ...;  /* Found */
RUN;
```
The hash table loads `lib.lookup` entirely into memory on the first iteration. `h.find()` performs
an in-memory lookup by `code`. This is lineage: `description` comes from `lib.lookup.description`
looked up by `lib.main.code`.

### 10.2 Anti-Patterns — Flag and Explain

| Anti-Pattern | Risk | Flag |
|---|---|---|
| Character variable length from first short assignment | Truncation of longer values | ⚠️ LENGTH NOT DECLARED |
| `IN (subquery)` where subquery may return NULL | Entire IN clause evaluates to NULL — zero rows returned | ⚠️ NULL IN SUBQUERY |
| `NOT IN (subquery)` where subquery may return NULL | NOT IN returns zero rows if any NULL in subquery | ⚠️ CRITICAL NULL NOT IN |
| MERGE without prior PROC SORT | Results are undefined; may ERROR or silently corrupt | ⚠️ MISSING PRE-SORT |
| RETAIN without group reset | Previous group's value bleeds into new group | ⚠️ RETAIN NO RESET |
| `MONOTONIC()` in a query with JOINs | Row numbers are not reliable after joins | ⚠️ MONOTONIC IN JOIN |
| Hard-coded date literals | Breaks at period boundaries without code changes | 📌 HARD-CODED DATE |
| Hard-coded year in macro variable | Must be updated manually each period | 📌 HARD-CODED PERIOD |
| Unresolved macro variable reference | Substitutes literal `&varname` text into SQL or code | ⚠️ UNRESOLVED MACRO VAR |
| PROC SQL with no WHERE on a large table | Full table scan — performance and correctness risk | ⚠️ NO FILTER ON LARGE TABLE |
| Multiple OUTPUT to same dataset in a DATA step | May create duplicate rows or unexpected interleaving | ⚠️ MULTIPLE OUTPUT |
| `CREATE VIEW` in PROC SQL | View re-executes on every reference — expensive if referenced repeatedly | ⚠️ VIEW NOT TABLE |
| Missing `NODUPKEY` when de-dup is expected | Duplicate rows silently pass through | ⚠️ POSSIBLE MISSING DEDUP |
| `NODUP` instead of `NODUPKEY` | Deduplicates on ALL columns, not just BY keys — rarely intended | ⚠️ NODUP VS NODUPKEY |
| LENGTH declared after first assignment | SAS ignores the LENGTH — type and length already set | ⚠️ LATE LENGTH DECLARATION |

---

## 11. Output Formats

### 11.1 Full Script Analysis

Structure output as:

**Section 1 — Program Overview**
Plain-language description (3–5 sentences). Libraries, macro variables, step inventory table.

**Section 2 — Step-by-Step Breakdown**
For each step: number, type, inputs, outputs, key operations, attributes created/modified.

**Section 3 — Dataset Flow**
Flow entries from Section 9.1 for each step. Classification of all datasets (external/intermediate/output).

**Section 4 — Attribute Dictionary**
Table of all attributes in each final output dataset: name, type, length, format, label,
definition type, expression/source, origin.

**Section 5 — Lineage Traces**
For each non-trivial attribute: full lineage chain using the notation from Section 8.4.

**Section 6 — Business Rules Catalog**
Every filter, condition, classification, and derivation rule in plain language.

**Section 7 — Flags and Risks**
All flagged items with flag type and explanation.

### 11.2 Targeted Attribute Lineage

When asked "where does attribute X come from?":

1. State where X appears in the final output dataset.
2. Provide the full lineage chain using ← notation.
3. Label every step, every transformation, every system boundary.
4. State the ultimate source (external system, static file, database table, or constant).
5. Flag any transformations that change semantics: filters that restrict population, aggregations
   that change granularity, formats that change display value, pivots that reshape structure.

### 11.3 Impact Analysis

When asked "what is affected if I change column X in table T?":

1. Find all scripts that read table T (from master_index Table Registry).
2. In each such script, find all attributes that derive from X directly or transitively.
3. For each downstream attribute, document the transformation chain from X to that attribute.
4. Identify any business rules, filters, or conditions that reference X.
5. List all final output tables that ultimately contain a value derived from X.

---

## 12. Knowledge Base Mode — Multi-Script Systems

Use this mode for any system with 5 or more SAS scripts. For 50+ scripts, this is mandatory.

**The principle**: analyze each script once, write structured artifacts, then serve all
downstream tasks from the artifacts. Raw `.sas` files are never re-read after Phase 1.

### 12.1 Output Directory Structure

```
/kb/
├── master_index.md                      ← Cross-script navigation index (built last)
├── scripts/
│   ├── script_001_<name>.md             ← One file per SAS script
│   ├── script_002_<name>.md
│   └── ...
├── tables/
│   └── _table_index.md                  ← All tables: producer script, consumer scripts
├── macros/
│   └── _macro_index.md                  ← All macros: definition, call sites, parameters
└── attributes/
    └── _attribute_index.md              ← All columns across all permanent output tables
```

**Naming convention**: `script_NNN_<original_script_name>.md`
- NNN = execution order, zero-padded to 3 digits.
- If execution order is unknown, use `script_<name>.md` and flag order as ⚠️ UNCONFIRMED.

### 12.2 Per-Script MD File Template

Every section is mandatory. Write `None` or `N/A` explicitly — absence is information.

````markdown
# Script: <filename>.sas
**Execution Order**: <NNN | Unknown ⚠️>
**Script Path**: <full path>
**Analyzed**: <date>
**Purpose**: <2–4 sentence plain-language summary of what this script does and why>

---

## 1. Environment

### Libraries
| Libref | Physical Path or Connection | Engine | Type |
|--------|-----------------------------|--------|------|
| WORK | Temporary SAS library | BASE | SAS temp |
| SRC | D:\data\source\ | BASE | SAS permanent |
| DB2LIB | DB2: server=mydb, schema=dbo | DB2 | Database |

### Filenames
| Fileref | Path / Endpoint | Purpose |
|---------|----------------|---------|
| RAWFILE | D:\inbound\rates.csv | Input flat file |

### Macro Variables Set in This Script
| Variable | Set By | Value or Expression | Scope | Consumed By |
|----------|--------|---------------------|-------|-------------|
| &report_year | %LET | 2024 | Global | Steps 03, 05 |
| &max_date | CALL SYMPUTX (Step 04) | MAX(date) from WORK.filtered | Global | Script 003 |

### Macro Variables Consumed (Set Elsewhere)
| Variable | Expected Value | Set In | Risk if Missing |
|----------|---------------|--------|-----------------|
| &adj_factor | Numeric multiplier | Script 001 | ⚠️ No default — step 05 breaks |

### Options
| Option | Value | Effect |
|--------|-------|--------|
| COMPRESS | YES | Compresses output datasets |
| OBS | MAX | Reads all rows (confirm not set to a test limit) |

---

## 2. Step Inventory

| Step | Type | Input(s) | Output(s) | Description |
|------|------|----------|-----------|-------------|
| 01 | LIBNAME | — | — | Maps SRC, OUT libraries |
| 02 | %LET | — | — | Sets &report_year=2024 |
| 03 | DATA | SRC.raw_txn | WORK.filtered | Filter active records for &report_year |
| 04 | PROC SORT | WORK.filtered | WORK.sorted | Sort by customer_id, date DESC; NODUPKEY |
| 05 | PROC SQL | WORK.sorted, SRC.customers | WORK.joined | LEFT JOIN + derive amount_adj, region_category |
| 06 | PROC MEANS | WORK.joined | WORK.summary | SUM revenue, units by region, product_type |
| 07 | DATA | WORK.summary | OUT.final_report | Format, label, drop internals |
| 08 | PROC EXPORT | OUT.final_report | D:\reports\out.csv | Export to CSV |

---

## 3. Tables

### Input Tables
| Table | Libref | Physical Source | Columns Used | Filter at Read | Notes |
|-------|--------|----------------|-------------|----------------|-------|
| SRC.raw_txn | SRC | D:\data\source\ | txn_id, customer_id, amount, status, date | status='ACTIVE' | 🔌 External source |
| SRC.customers | SRC | D:\data\source\ | customer_id, customer_name, region | None | 🔌 External source |

### Output Tables
| Table | Libref | Permanent? | Producing Step | Consumed By | Description |
|-------|--------|------------|----------------|-------------|-------------|
| WORK.filtered | WORK | No | 03 | 04 | Active txns for reporting period |
| WORK.sorted | WORK | No | 04 | 05 | De-duped, sorted |
| WORK.joined | WORK | No | 05 | 06 | Txn + customer, with derived cols |
| WORK.summary | WORK | No | 06 | 07 | Aggregated by region/product |
| OUT.final_report | OUT | Yes | 07 | (terminal) | Final deliverable |

---

## 4. Macros

### Defined in This Script
| Macro | Parameters | Purpose | SAS Steps Generated |
|-------|-----------|---------|---------------------|
| %calc_tax(rate=, base=) | rate (required), base (required) | Computes tax amount | Inline arithmetic — no steps |

### Called in This Script
| Macro | Called at Step | Parameters Passed | Defined In |
|-------|--------------|------------------|-----------|
| %load_lookup | 02 | inlib=SRC, outlib=WORK | utils/lookup_helper.sas ⚠️ external |
| %calc_tax | 05 | rate=&tax_rate, base=net_amount | This script |

---

## 5. Step Detail

*(One subsection per step. Copy and fill this block for every step.)*

### Step 03 — DATA: Filter Raw Transactions
**Input**: `SRC.raw_txn`
**Output**: `WORK.filtered`
**Row Filter**: `status = 'ACTIVE'` AND `YEAR(date) = &report_year` (=2024 📌) applied at SET
**Column Filter**: KEEP= txn_id customer_id amount status date
**Subsetting IF**: `IF amount > 0;` — drops zero/negative/missing amounts

```sas
DATA WORK.filtered;
  SET SRC.raw_txn (KEEP=txn_id customer_id amount status date
                   WHERE=(status='ACTIVE' AND YEAR(date)=&report_year));
  IF amount > 0;
RUN;
```

**Attributes Produced**:
| Attribute | Type | Len | Definition | Source |
|-----------|------|-----|-----------|--------|
| txn_id | Char | $20 | Pass-through | SRC.raw_txn.txn_id |
| customer_id | Char | $15 | Pass-through | SRC.raw_txn.customer_id |
| amount | Num | 8 | Pass-through (filtered > 0) | SRC.raw_txn.amount |
| status | Char | $1 | Pass-through (filtered = 'ACTIVE') | SRC.raw_txn.status |
| date | Num (Date) | 8 | Pass-through | SRC.raw_txn.date |

**Flags**: 📌 &report_year hard-coded to 2024 via %LET Step 02

---

### Step 05 — PROC SQL: Join + Derive
**Input**: `WORK.sorted` (alias t), `SRC.customers` (alias c)
**Output**: `WORK.joined`
**Join**: LEFT JOIN on t.customer_id = c.customer_id
**Row Filter**: `t.date BETWEEN '01JAN2024'D AND '31DEC2024'D` 📌
**Granularity**: One row per transaction (txn_id)

```sql
CREATE TABLE WORK.joined AS
SELECT
    t.txn_id,
    t.customer_id,
    c.customer_name,
    c.region,
    t.date                      AS transaction_date,
    t.amount,
    t.amount * &adj_factor      AS amount_adj,
    CALCULATED amount_adj * 0.05 AS estimated_fee,
    CASE WHEN c.region IN ('NORTH','SOUTH') THEN 'Domestic'
         WHEN c.region IN ('EAST','WEST')   THEN 'Cross-Region'
         ELSE 'Unknown' END     AS region_category
FROM WORK.sorted t
LEFT JOIN SRC.customers c ON t.customer_id = c.customer_id
WHERE t.date BETWEEN '01JAN2024'D AND '31DEC2024'D
ORDER BY c.region, t.date;
```

**Attributes Produced**:
| Attribute | Type | Definition | Source Columns | Flags |
|-----------|------|-----------|----------------|-------|
| txn_id | Char | Pass-through | WORK.sorted.txn_id | |
| customer_id | Char | Pass-through | WORK.sorted.customer_id | |
| customer_name | Char | Pass-through | SRC.customers.customer_name | ⚠️ NULL if no customer match |
| region | Char | Pass-through | SRC.customers.region | ⚠️ NULL if no customer match |
| transaction_date | Num | Renamed: t.date | WORK.sorted.date | |
| amount | Num | Pass-through | WORK.sorted.amount | |
| amount_adj | Num | amount × &adj_factor (=1.15) | amount, &adj_factor | 🔧📌 |
| estimated_fee | Num | CALCULATED amount_adj × 0.05 | amount_adj | |
| region_category | Char | CASE WHEN region IN (...) | region | ⚠️ 'Unknown' when region is NULL |

*(Continue for every step in the script)*

---

## 6. Attribute Dictionary — Final Output: OUT.final_report

| # | Attribute | Type | Len | Format | Label | Definition Type | Expression / Source | Origin |
|---|-----------|------|-----|--------|-------|----------------|---------------------|--------|
| 1 | txn_id | Char | $20 | | Transaction ID | Pass-through | SRC.raw_txn.txn_id | This script |
| 2 | customer_id | Char | $15 | | Customer ID | Pass-through | SRC.raw_txn.customer_id | This script |
| 3 | customer_name | Char | $100 | | Customer Name | Pass-through | SRC.customers.customer_name | This script |
| 4 | region | Char | $20 | | Region | Pass-through | SRC.customers.region | This script |
| 5 | transaction_date | Num | 8 | DATE9. | Transaction Date | Renamed pass-through | SRC.raw_txn.date | This script |
| 6 | amount | Num | 8 | COMMA12.2 | Transaction Amount | Pass-through (filtered > 0) | SRC.raw_txn.amount | This script |
| 7 | amount_adj | Num | 8 | COMMA12.2 | Adjusted Amount | Derived-arithmetic | amount × 1.15 | This script |
| 8 | estimated_fee | Num | 8 | COMMA12.2 | Estimated Fee | Derived-arithmetic | amount_adj × 0.05 | This script |
| 9 | region_category | Char | $20 | | Region Category | Derived-conditional | CASE WHEN region IN (...) | This script |

---

## 7. Lineage Summary

```
OUT.final_report.amount_adj
  ← [Step 07 DATA] Pass-through from WORK.summary.amount_adj
      ← [Step 06 PROC MEANS] SUM of WORK.joined.amount_adj by region/product_type
          ← [Step 05 PROC SQL] Derived: amount × &adj_factor
              ← amount ← [Step 03 DATA] Pass-through from SRC.raw_txn.amount (filtered > 0)
                  🔌 SRC.raw_txn ← D:\data\source\ (external SAS library)
              ← &adj_factor = 1.15 🔧📌 %LET Step 02

OUT.final_report.region_category
  ← [Step 07 DATA] Pass-through from WORK.summary.region_category
      ← [Step 06 PROC MEANS] Pass-through (CLASS variable, not aggregated)
          ← [Step 05 PROC SQL] CASE WHEN region IN ('NORTH','SOUTH') → 'Domestic' ...
              ← region ← [Step 05 PROC SQL] LEFT JOIN SRC.customers ON customer_id
                  ⚠️ region = NULL when no customer match → region_category = 'Unknown'
                  🔌 SRC.customers ← D:\data\source\ (external SAS library)
```

---

## 8. Business Rules Catalog

| Rule # | Type | Step | Plain-Language Rule | Attributes Affected |
|--------|------|------|---------------------|---------------------|
| R001 | Row filter | 03 | Only ACTIVE status records are processed | All downstream |
| R002 | Row filter | 03 | Only year 2024 records (via &report_year) 📌 | All downstream |
| R003 | Row filter | 03 | Only positive amount records | amount and all derivatives |
| R004 | De-dup | 04 | One record per customer_id per date — keep most recent (sort DESC) | All fact attributes |
| R005 | Join | 05 | LEFT JOIN — all transactions kept regardless of customer match | customer_name, region may be NULL |
| R006 | Row filter | 05 | Date range 01JAN2024–31DEC2024 📌 (partially redundant with R002) | All |
| R007 | Derivation | 05 | Adjusted amount = amount × 1.15 📌 | amount_adj, estimated_fee |
| R008 | Classification | 05 | NORTH/SOUTH → Domestic; EAST/WEST → Cross-Region; else Unknown | region_category |

---

## 9. Flags and Risks

| # | Type | Step | Description | Action Needed |
|---|------|------|-------------|---------------|
| F01 | 📌 Hard-coded | 02 | &report_year = 2024 | Update annually |
| F02 | 📌 Hard-coded | 05 | Date range '01JAN2024'D–'31DEC2024'D overlaps with R002; drift risk | Parameterize |
| F03 | ⚠️ NULL risk | 05 | LEFT JOIN — customer_name and region NULL for unmatched txns | Validate customer master coverage |
| F04 | 🔧 Macro dep | 05 | &adj_factor must be set before this script — no default | Add %IF check or default |
| F05 | ⚠️ External file | 02 | %load_lookup references utils/lookup_helper.sas — must exist | Confirm file path |
````

---

### 12.3 Master Index Template

Build this last, after all per-script `.md` files are complete.

````markdown
# SAS System Master Index
**System**: <system name>
**Scripts Total**: <N>
**Generated**: <date>
**KB Root**: /kb/

---

## 1. Script Execution Order

| Order | MD File | Script | Purpose | Key Outputs |
|-------|---------|--------|---------|-------------|
| 001 | scripts/script_001_load_customers.md | load_customers.sas | Load and clean customer master | OUT.dim_customer |
| 002 | scripts/script_002_transform_orders.md | transform_orders.sas | Enrich raw orders | OUT.fact_orders |
| 003 | scripts/script_003_aggregate_revenue.md | aggregate_revenue.sas | Revenue summary by region | OUT.revenue_summary |

---

## 2. Table Registry

| Table | Libref | Type | Permanent | Created By | Read By | Physical Location |
|-------|--------|------|-----------|-----------|---------|-------------------|
| SRC.raw_txn | SRC | External Source | Yes | (external) | 001, 003 | D:\data\source\ |
| SRC.customers | SRC | External Source | Yes | (external) | 001, 002 | D:\data\source\ |
| OUT.dim_customer | OUT | Final Output | Yes | 001 | 002, 003 | D:\data\output\ |
| OUT.fact_orders | OUT | Final Output | Yes | 002 | 003 | D:\data\output\ |
| OUT.revenue_summary | OUT | Terminal Output | Yes | 003 | (none) | D:\data\output\ |

---

## 3. Attribute Registry

| Table | Attribute | Type | Definition Type | Origin Script | Source Column(s) |
|-------|-----------|------|----------------|---------------|------------------|
| OUT.dim_customer | customer_id | Char | Pass-through | 001 | SRC.customers.customer_id |
| OUT.dim_customer | region_category | Char | Derived-conditional | 001 | SRC.customers.region |
| OUT.fact_orders | amount_adj | Num | Derived-arithmetic | 002 | SRC.raw_orders.amount × 1.15 |
| OUT.revenue_summary | total_revenue | Num | Derived-aggregate | 003 | OUT.fact_orders.amount_adj SUM() |

---

## 4. Macro Registry

| Macro | Defined In | Called In | Parameters | Purpose |
|-------|-----------|----------|-----------|---------|
| %load_lookup | utils/lookup_helper.sas ⚠️ | 001, 002 | inlib=, outlib= | Loads reference lookups |
| %calc_tax | 002 | 002, 003 | rate=, base= | Tax calculation |

---

## 5. Macro Variable Registry

| Variable | Set In | Set By | Value / Expression | Used In | Notes |
|----------|--------|--------|-------------------|---------|-------|
| &report_year | 001 | %LET | 2024 | 001, 002, 003 | 📌 Update annually |
| &adj_factor | 002 | %LET | 1.15 | 002 | 📌 Hard-coded multiplier |
| &max_date | 002 | CALL SYMPUTX | MAX(date) from WORK.filtered | 003 | 🔧 Cross-script timing dependency |
| &env | controller.sas | %LET | PROD / DEV | All | Controls library paths |

---

## 6. External Source Registry

| Source | Type | Connection / Path | Tables | Consumed By |
|--------|------|------------------|--------|-------------|
| D:\data\source\ | SAS Library (SRC) | Physical folder | raw_txn, customers, products | 001, 002 |
| Oracle: financedb | Pass-through SQL | schema=finance | gl_transactions, gl_accounts | 003 |
| D:\inbound\rates.csv | Flat File | FILENAME RAWFILE | (PROC IMPORT → WORK.rates) | 001 |

---

## 7. Execution Dependency Map

| Script | Must Run After | Dependency Reason |
|--------|---------------|------------------|
| 002 | 001 | Reads OUT.dim_customer produced by 001 |
| 003 | 001, 002 | Reads OUT.dim_customer and OUT.fact_orders |
| 003 | 002 | Consumes &max_date set by 002 (timing dependency) |

---

## 8. Business Rules — Master List

| Rule ID | Script | Step | Rule | Attributes Affected |
|---------|--------|------|------|---------------------|
| R001-R001 | 001 | 03 | status = 'ACTIVE' only | All downstream |
| R001-R002 | 001 | 03 | amount > 0 only | amount and all derivatives |
| R001-R008 | 001 | 05 | NORTH/SOUTH → Domestic; EAST/WEST → Cross-Region | region_category |
| R002-R003 | 002 | 04 | Most recent order per customer (NODUPKEY, date DESC) | All fact_orders cols |

---

## 9. Flags and Risks — Master List

| Flag ID | Script | Type | Description |
|---------|--------|------|-------------|
| F001-F01 | 001 | 📌 | &report_year = 2024 — update annually |
| F001-F03 | 001 | ⚠️ | LEFT JOIN — customer_name/region may be NULL |
| F002-F01 | 002 | 🔧 | &max_date timing dependency — 003 breaks if 002 hasn't run |

---

## 10. Unresolved References

| Item | Referenced In | Type | Notes |
|------|-------------|------|-------|
| utils/lookup_helper.sas | 001, 002 (%load_lookup) | %INCLUDE dependency | File not in analyzed script set ⚠️ |
````

---

### 12.4 Phase 1 — Build the Knowledge Base

**Execute in this exact sequence:**

1. **Inventory** — List all `.sas` files. Determine execution order by looking for: a job
   scheduler config, a master controller macro, `%INCLUDE` chains, or LIBNAME/table dependencies.
   Ask the user if order is ambiguous.

2. **Analyze in execution order** — Process upstream scripts first so table lineage is already
   documented when downstream scripts reference those tables. For each script:
   a. Apply Sections 1–10 of this skill fully.
   b. Write the per-script `.md` using the template in Section 12.2.
   c. Save to `/kb/scripts/script_NNN_<name>.md`.

3. **Build sub-indexes** — After all per-script files are complete, aggregate:
   - `/kb/tables/_table_index.md` — all table registry entries from all scripts.
   - `/kb/macros/_macro_index.md` — all macro registry entries.
   - `/kb/attributes/_attribute_index.md` — all attribute dictionary entries from permanent tables.

4. **Write master_index.md** — Consolidate all registries into `/kb/master_index.md`.

5. **Validate** — For every input table referenced in any script, confirm it either:
   - Appears as a terminal output of another script in the Table Registry, OR
   - Is listed in the External Source Registry with a confirmed physical location.
   - If neither: flag as ⚠️ `UNRESOLVED SOURCE` in master_index.md.

6. **Check cross-script macro variable dependencies** — For every macro variable in the
   Macro Variable Registry that is set in one script and consumed in another, confirm the
   producing script runs first. Flag any timing violations.

---

### 12.5 Phase 2 — Query the Knowledge Base

**For all downstream tasks, follow this protocol. Never re-read raw `.sas` files.**

**Step 1 — Always open `master_index.md` first.**
Use it to answer: which scripts are involved? Which `.md` files do I need? What dependencies exist?

**Step 2 — Open the minimum necessary per-script `.md` files.**

| Task | Which sections to read in the MD files |
|------|----------------------------------------|
| Attribute lineage | Lineage Summary + Step Detail of producing script; recurse via Table Registry |
| Impact analysis | Table Registry → find all consumer scripts → their Attribute Dictionary and Business Rules |
| Mapping sheet | Attribute Dictionary from each relevant script |
| Business rule audit | Business Rules Catalog from all scripts (or master list) |
| Dependency analysis | Execution Dependency Map in master_index.md |
| Risk review | Flags from all scripts (or master list) |

**Step 3 — Recurse only as needed.**
If a lineage trace reaches a table listed as produced by another script, open that script's
`.md` and continue from its Lineage Summary. Stop at External Sources.

**Step 4 — Never open more `.md` files than the master index identifies as relevant.**

---

### 12.6 Cross-Reference Naming Conventions

These conventions must be applied consistently across all `.md` files:

| Reference Type | Format | Example |
|---|---|---|
| Table | `LIBREF.DATASETNAME` (uppercase) | `OUT.FACT_ORDERS` |
| Attribute | `LIBREF.DATASETNAME.ATTRIBUTE` (uppercase) | `OUT.FACT_ORDERS.AMOUNT_ADJ` |
| External DB table | `DB:<schema>.<table>` | `DB:finance.gl_transactions` |
| Script reference | MD filename | `scripts/script_002_transform_orders.md` |
| Business rule | `R<NNN>-R<nn>` | `R001-R03` (script 001, rule 3) |
| Flag | `F<NNN>-F<nn>` | `F002-F01` (script 002, flag 1) |
| Macro variable | `&VARNAME` (uppercase) | `&REPORT_YEAR` |
| Macro | `%MACRONAME` (uppercase) | `%LOAD_LOOKUP` |

---

## 13. Self-Refinement Protocol

The skill improves through use. After every analysis session, the agent must update
`/kb/skill_learnings.md` — the persistent learning log for this system. If this file does not
exist, create it. Never delete entries — append only.

This is not optional. Every session produces something worth recording.

### 13.1 What to Record

After completing any analysis task, reflect on the following questions and write findings:

**Patterns encountered:**
- Did this script use a SAS pattern not covered in Sections 3–6? Describe it precisely.
- Did a known pattern appear in an unexpected combination? Describe the variation.
- Did a macro generate SAS code in a way that was harder than expected to unroll? Describe the technique.

**Lineage complexity:**
- Were there multi-hop lineage chains that were particularly deep or branched? Note the pattern.
- Were there macro variable chains that crossed more than 2 scripts? Document the chain.
- Were there RETAIN-based attributes that had complex reset logic? Document it.
- Were there attributes that appeared simple but had hidden dependencies (e.g., sort-order dependency)?

**Anti-patterns found:**
- Were new anti-patterns encountered that are not listed in Section 10.2? Add them.
- Were known anti-patterns found in a new form? Describe the variant.

**Ambiguities and resolutions:**
- Were there cases where the correct interpretation was unclear? What was the resolution?
- Were there cases where the methodology needed to be adapted? How?

**Errors to avoid:**
- Were there cases where an initial interpretation was wrong? What was the mistake and correction?
- Were there cases where a column was assumed to be a pass-through but was actually derived?
- Were there cases where a join was assumed to be INNER but was effectively a LEFT (or vice versa)?

**Efficiency gains:**
- Were there patterns that allowed faster analysis than the standard methodology? Document them.
- Were there sections of the knowledge base template that were consistently unused for this system?
  Note them for future streamlining.

### 13.2 Skill Learnings File Format

````markdown
# Skill Learnings Log
**System**: <system name>
**KB Root**: /kb/

---

## Entry: <date> — Session: <brief task description>

### Patterns Encountered
- <pattern description>

### New Anti-Patterns
- <anti-pattern>: <description, risk, how to detect>

### Lineage Complexity Notes
- <note>

### Ambiguities and Resolutions
- Situation: <what was ambiguous>
  Resolution: <how it was resolved>
  Principle: <the general rule to apply next time>

### Errors Corrected
- Initial assumption: <what was assumed>
  Correct interpretation: <what it actually was>
  Cause: <why the error occurred>
  Prevention: <what to check to avoid this>

### Efficiency Notes
- <observation>

---

*(Next entry below — append only)*
````

### 13.3 Applying Learnings in Future Sessions

At the start of every new session on a system that has a `skill_learnings.md`:

1. **Read `skill_learnings.md` before reading `master_index.md`.**
2. Extract the "Errors Corrected" and "Ambiguities and Resolutions" entries — these define
   system-specific interpretation rules that override default methodology.
3. Extract the "Patterns Encountered" entries — these prime the analysis for patterns known
   to appear in this system.
4. Apply all "New Anti-Patterns" as additional checks during analysis.

### 13.4 Skill-Level vs System-Level Learnings

Two types of learnings exist:

| Type | What it is | Where recorded |
|---|---|---|
| **System-specific** | Patterns, rules, or quirks specific to this particular SAS system | `skill_learnings.md` in the system's `/kb/` folder |
| **Skill-level** | A gap or error in the SKILL.md itself — the methodology is incomplete or wrong | Flag explicitly with `SKILL UPDATE NEEDED:` prefix in skill_learnings.md |

When a `SKILL UPDATE NEEDED` entry is identified, the agent should:
1. Record the gap in `skill_learnings.md`.
2. Propose the exact addition or correction to SKILL.md to the user.
3. If authorized, update SKILL.md directly and note the version change.

### 13.5 Refinement Triggers

The following events always trigger a skill_learnings.md update entry:

- A SAS construct was encountered that this skill did not cover.
- A lineage trace produced a result that was initially incorrect and had to be revised.
- A new anti-pattern was found that caused or could cause data quality issues.
- The per-script `.md` template was found to be missing a field that would have been useful.
- A downstream task (mapping sheet, impact analysis) revealed a gap in the knowledge base structure.
- A cross-script dependency was discovered that was not captured in the master_index.

---

## Quick Reference

### SAS SQL vs Standard SQL

| Feature | Standard SQL | SAS PROC SQL |
|---|---|---|
| Forward-reference alias | Not allowed | `CALCULATED alias` |
| Row number | `ROW_NUMBER() OVER (...)` | `MONOTONIC()` — unreliable in joins |
| Character → date | `CAST('2024-01-01' AS DATE)` | `INPUT('01JAN2024', DATE9.)` |
| Date → character | `TO_CHAR(dt, 'fmt')` | `PUT(dt, DATE9.)` |
| Date literal | `DATE '2024-01-01'` | `'01JAN2024'D` |
| Missing/NULL check | `IS NULL` | `IS NULL` or `IS MISSING` |
| Limit output rows | `LIMIT n` / `TOP n` | `OUTOBS=n` on PROC SQL statement |
| Macro variable at runtime | N/A | `SYMGET('var')` |
| External DB query | N/A | `CONNECTION TO dbname (native SQL)` |
| Dataset options on FROM | N/A | `FROM lib.ds (WHERE= KEEP= RENAME=)` |
| Suppress output | N/A | `PROC SQL NOPRINT;` |

### DATA Step Join Type Quick Reference

| IF Condition on MERGE | Join Type |
|---|---|
| `IF a;` | Left join |
| `IF b;` | Right join |
| `IF a AND b;` | Inner join |
| `IF a OR b;` | Full outer join |
| `IF NOT b;` | Left anti-join |
| `IF NOT a AND b;` | Right anti-join |
| *(no IF)* | Full outer join (SAS default) |

### Attribute Classification Quick Reference

| Class | Key Signal |
|---|---|
| Pass-through | Column name appears unchanged from SET/FROM source |
| Renamed | `RENAME=` option or `AS new_name` in SELECT |
| Derived-arithmetic | `=` with operators, no IF/CASE |
| Derived-conditional | IF/ELSE, SELECT/WHEN, CASE/WHEN |
| Derived-aggregate | SUM, COUNT, AVG, etc. |
| Derived-lookup | `PUT(var, fmt.)` or joined from reference table |
| Macro-injected | `&macro_var` substituted into expression |
| Constant | Literal value — no column reference |

### Flag and Notation Key

| Symbol | Meaning |
|---|---|
| `←` | Lineage direction: output derived from source on right |
| `🔌` | External system boundary |
| `📌` | Hard-coded constant — should be parameterized |
| `🔧` | Macro variable dependency |
| `⚠️` | Risk, anomaly, or anti-pattern |
| `[Step N TYPE]` | Step reference in lineage chain |

---

## 14. Validation Checks

Run these checks at three points: (A) after analyzing each individual step, (B) after completing
a full per-script `.md` file, and (C) after completing the full `master_index.md`. Each check
that fails must either be resolved immediately or escalated using the Doubt Escalation Protocol
in Section 15 before proceeding.

---

### 14.1 Step-Level Checks (Run After Each Step)

#### V-S01 — All input datasets resolved
Every dataset referenced in `SET`, `MERGE`, `UPDATE`, `PROC SQL FROM`, `DATA=`, `BASE=` must be
either (a) listed as an output of a prior step in this script, (b) confirmed as an external
source via LIBNAME, or (c) escalated as `⚠️ UNRESOLVED SOURCE`.

**Fail condition**: A dataset reference has no LIBNAME mapping and no producing step.
**Action**: Escalate → Section 15 Doubt Type D-02.

#### V-S02 — All columns in output traced to a source
Every column in the step's output must be classified (Section 7.2) and traced to at least one
input column, constant, macro variable, or system variable. No column should appear in the
output without an identified origin.

**Fail condition**: A column is in `KEEP=` or the SELECT list but its derivation is not clear.
**Action**: Escalate → Section 15 Doubt Type D-01.

#### V-S03 — JOIN type confirmed
For every MERGE or PROC SQL JOIN, the join type must be explicitly stated and verified against
the code. `IN=` flag conditions (DATA step) and JOIN keywords (PROC SQL) must match the
stated join type.

**Fail condition**: No `IF` condition after MERGE (SAS default is full outer — often not intended).
Missing `IN=` variables when join filtering is expected.
**Action**: Flag ⚠️ UNCONFIRMED JOIN TYPE. Escalate → Section 15 Doubt Type D-03.

#### V-S04 — Pre-sort verified for MERGE
Every MERGE must have a PROC SORT (or verified index) on the BY key immediately prior to it,
covering all datasets being merged.

**Fail condition**: MERGE with BY but no PROC SORT in preceding steps on those variables.
**Action**: Flag ⚠️ MISSING PRE-SORT. Escalate → Section 15 Doubt Type D-04.

#### V-S05 — RETAIN variables have reset logic
Every `RETAIN`ed variable that is used in a group-level accumulation must have a
`IF FIRST.var THEN retained_var = <init>;` reset pattern, or an equivalent reset mechanism.

**Fail condition**: `RETAIN` without any `FIRST.` reset — value bleeds across group boundaries.
**Action**: Flag ⚠️ RETAIN NO RESET. Escalate → Section 15 Doubt Type D-05.

#### V-S06 — Macro variables in code are resolved
Every `&macro_var` reference in the step must be traced to a `%LET`, `%GLOBAL`,
`CALL SYMPUTX`, or `SELECT INTO :var` assignment. If the assignment is in a prior script
(cross-script macro dependency), it must be in the Macro Variable Registry.

**Fail condition**: A `&var` reference with no visible assignment in this script or the
Macro Variable Registry.
**Action**: Flag ⚠️ UNRESOLVED MACRO VAR. Escalate → Section 15 Doubt Type D-06.

#### V-S07 — NODUPKEY/NODUP intent confirmed
When PROC SORT uses `NODUPKEY` or `NODUP`, confirm which is intended:
- `NODUPKEY`: de-duplicates on BY key columns only. The "first" record is kept based on
  the sort order. Verify the sort direction matches the intended kept record.
- `NODUP`: de-duplicates on ALL columns. Rarely what's intended when a key de-dup is expected.

**Fail condition**: `NODUP` used where `NODUPKEY` appears to be the intent (BY key present,
full-row de-dup is unexpected).
**Action**: Flag ⚠️ NODUP VS NODUPKEY. Escalate → Section 15 Doubt Type D-07.

#### V-S08 — Character variable lengths declared
Every character variable that is assigned a string value in a DATA step must have a `LENGTH`
declaration before its first assignment, or its length must be demonstrably correct from
the first assignment value.

**Fail condition**: Character variable assigned a short string on first use; later assigned
longer values; no `LENGTH` declaration.
**Action**: Flag ⚠️ LENGTH NOT DECLARED. Escalate → Section 15 Doubt Type D-08.

#### V-S09 — CALCULATED references unrolled
Every `CALCULATED col` in a PROC SQL SELECT must be traced back to the column alias it
references and the full expression documented. Chains of CALCULATED references (A refers to B
which refers to C) must be fully unrolled.

**Fail condition**: `CALCULATED col` where `col` is not defined earlier in the same SELECT list.
**Action**: Flag ⚠️ BROKEN CALCULATED REFERENCE. Escalate → Section 15 Doubt Type D-09.

#### V-S10 — NOT IN subquery NULL risk checked
Every `NOT IN (subquery)` must be checked: if the subquery can return a NULL value, the
entire NOT IN condition will return no rows, silently dropping all data.

**Fail condition**: `NOT IN (SELECT col FROM ...)` where `col` has no NOT NULL constraint and
no `WHERE col IS NOT NULL` in the subquery.
**Action**: Flag ⚠️ CRITICAL NULL NOT IN. Escalate → Section 15 Doubt Type D-10.

---

### 14.2 Script-Level Checks (Run After Completing Per-Script MD)

#### V-P01 — Step inventory is complete
Count the steps in the actual `.sas` file. Count the steps in the Step Inventory table in the
`.md` file. They must match. Every `DATA`, `PROC`, `%MACRO` definition, `%macro_call`,
`LIBNAME`, `FILENAME`, `%LET`, and `OPTIONS` statement is a step entry.

**Fail condition**: Step counts do not match, or a step type is missing from the inventory.
**Action**: Re-scan the script. If still unclear, escalate → Section 15 Doubt Type D-11.

#### V-P02 — All final output columns are in the Attribute Dictionary
The Attribute Dictionary must list every column that exists in each permanent output dataset.
No columns may be undocumented. Verify against `KEEP=` lists, `DROP=` lists, and the last
DATA step / PROC SQL that writes each output table.

**Fail condition**: A column appears in the output dataset (inferred from KEEP= or not in DROP=)
but is not in the Attribute Dictionary.
**Action**: Escalate → Section 15 Doubt Type D-01.

#### V-P03 — Lineage for every derived attribute is complete
Every attribute classified as Derived, Aggregate, Conditional, Lookup, or Macro-injected
must have a complete lineage chain in the Lineage Summary section. The chain must terminate
at an External Source, Constant, or System variable. No broken chains.

**Fail condition**: Lineage chain for an attribute has no terminal node — it ends at an
intermediate dataset with no further trace.
**Action**: Escalate → Section 15 Doubt Type D-12.

#### V-P04 — Business rules catalog covers all filters and conditions
Every `WHERE`, subsetting `IF`, `HAVING`, `IN (...)`, `BETWEEN`, `CASE WHEN`, `SELECT/WHEN`,
and `IF/ELSE` in the script must appear as a row in the Business Rules Catalog.
No conditional logic may be undocumented.

**Fail condition**: A filter or conditional is in the step code but not in the Business Rules Catalog.
**Action**: Add the missing rule. If the intent is unclear, escalate → Section 15 Doubt Type D-13.

#### V-P05 — Macro calls to external files flagged
Every `%INCLUDE` statement and every macro call where the macro is defined in an external
file (not in this script) must be flagged in the Macros section with its source file path.
If the source file is not in the analyzed script set, flag ⚠️ EXTERNAL DEPENDENCY.

**Fail condition**: A macro is called but not defined anywhere in this script, with no notation
of where it is defined.
**Action**: Escalate → Section 15 Doubt Type D-14.

#### V-P06 — All %INCLUDE files noted
Every `%INCLUDE 'path.sas';` must be listed as an external dependency. If the included file
is in the script set being analyzed, confirm it has (or will have) its own `.md` file.
If not, flag it for separate analysis.

**Fail condition**: `%INCLUDE` present but not documented.
**Action**: Add to External Dependencies. Escalate if content is unknown → Section 15 Doubt Type D-14.

#### V-P07 — Cross-script macro variable dependencies recorded
Every macro variable that is set in this script and consumed in another, or consumed in this
script but set in another, must be recorded in the Macro Variables section with the cross-script
reference. These entries feed the Master Index Macro Variable Registry.

**Fail condition**: A macro variable appears to be set in this script but is consumed by a step
in a different script, with no cross-script note.
**Action**: Flag 🔧 CROSS-SCRIPT MACRO VAR. Escalate → Section 15 Doubt Type D-15.

#### V-P08 — Hard-coded values inventoried
All hard-coded dates, years, numeric constants, and string literals that represent business
parameters (not structural code) must be flagged 📌 in the relevant step detail and listed
in the Flags and Risks section.

**Fail condition**: A literal value is used as a business parameter (e.g., a year, a rate,
a status code used as a filter threshold) without a 📌 flag.
**Action**: Add the flag. If it is unclear whether the value is a business parameter or a
structural constant, escalate → Section 15 Doubt Type D-16.

---

### 14.3 Master Index Checks (Run After Completing master_index.md)

#### V-M01 — Every input table is accounted for
For every table in the Table Registry marked as an input (consumed by any script), it must
either be marked as:
- Produced by another script in the registry, OR
- Listed in the External Source Registry with a confirmed physical location/connection.

**Fail condition**: A table is consumed but has no producer script and no External Source entry.
**Action**: Flag ⚠️ UNRESOLVED SOURCE. Escalate → Section 15 Doubt Type D-02.

#### V-M02 — Execution dependency map is complete
For every script that reads a table produced by another script, there must be a dependency
edge in the Execution Dependency Map. For every cross-script macro variable dependency,
there must be a dependency edge.

**Fail condition**: Script B reads `OUT.dim_customer` produced by Script A, but no dependency
edge A → B exists in the map.
**Action**: Add the missing edge. If execution order is ambiguous, escalate → Section 15 Doubt Type D-17.

#### V-M03 — No circular dependencies
The Execution Dependency Map must be a Directed Acyclic Graph (DAG). No script may
depend on itself, and no cycle may exist (A → B → C → A).

**Fail condition**: A circular dependency is detected.
**Action**: Escalate immediately → Section 15 Doubt Type D-17. Do not attempt to resolve
without user input — circular dependencies indicate either a system design issue or an
error in the dependency map.

#### V-M04 — Terminal outputs are confirmed
Every table listed in the Table Registry must be either consumed by at least one script or
marked as a terminal output. Unconsumable tables that are not marked as terminal outputs
may indicate dead code or stale scripts.

**Fail condition**: A permanent output table is never read by any other script and is not
marked as a terminal output.
**Action**: Flag ⚠️ POSSIBLE DEAD OUTPUT. Escalate → Section 15 Doubt Type D-18.

#### V-M05 — Attribute Registry covers all permanent output tables
Every permanent output table in the Table Registry must have its full attribute set listed
in the Attribute Registry. No permanent output table may have zero attribute entries.

**Fail condition**: A permanent output table exists in the Table Registry with no
corresponding rows in the Attribute Registry.
**Action**: Open the producing script's `.md` Attribute Dictionary and populate the entries.
If the producing script's `.md` is incomplete, escalate → Section 15 Doubt Type D-01.

#### V-M06 — Macro variable timing is safe
For every cross-script macro variable in the Macro Variable Registry:
- The producing script must have a lower execution order number than all consuming scripts.
- If a macro variable is produced by a CALL SYMPUTX inside a DATA step, no step within
  the SAME script can consume it before that DATA step completes.

**Fail condition**: Consuming script has a lower order number than producing script.
Or macro variable is consumed in the same step that sets it.
**Action**: Flag 🔧 MACRO VAR TIMING VIOLATION. Escalate → Section 15 Doubt Type D-15.

#### V-M07 — Unresolved references list is current
The Unresolved References section of master_index.md must be reviewed and confirmed current.
Every item listed must still be unresolved (not since resolved in a later session).
Every newly discovered unresolved reference from this session must be added.

**Fail condition**: A reference was resolved in this session but still appears as unresolved,
or a newly discovered unresolved reference was not added.
**Action**: Update the Unresolved References section before closing the session.

---

### 14.4 Validation Status Notation

When recording validation results in `.md` files, use this notation:

| Status | Symbol | Meaning |
|---|---|---|
| Passed | `✅` | Check ran and passed — no issues found |
| Failed — resolved | `🔧✅` | Check failed; resolved within session |
| Failed — escalated | `❓` | Check failed; escalated to user (see doubt log) |
| Failed — flagged | `⚠️` | Check failed; risk documented; not yet resolved |
| Skipped — N/A | `—` | Check not applicable for this step/script |

Add a validation summary block to the end of each per-script `.md`:

````markdown
## 10. Validation Summary

| Check | Status | Notes |
|-------|--------|-------|
| V-S01 All inputs resolved | ✅ | All datasets confirmed |
| V-S02 All output columns traced | 🔧✅ | customer_name initially missed — added |
| V-S03 JOIN type confirmed | ✅ | LEFT JOIN on customer_id verified |
| V-S04 Pre-sort verified | ✅ | PROC SORT Step 04 confirmed |
| V-S05 RETAIN reset logic | — | No RETAIN used in this script |
| V-S06 Macro vars resolved | ❓ | &adj_factor source unclear — escalated D-06 |
| V-S07 NODUPKEY intent | ✅ | NODUPKEY on customer_id + date confirmed |
| V-S08 Char lengths declared | ⚠️ | customer_id length inferred — F05 flagged |
| V-S09 CALCULATED unrolled | ✅ | estimated_fee chain fully unrolled |
| V-S10 NOT IN NULL risk | — | No NOT IN used in this script |
| V-P01 Step inventory complete | ✅ | 8 steps — matches script |
| V-P02 Attribute Dict complete | ✅ | 9 attributes documented |
| V-P03 Lineage complete | ✅ | All derived attributes traced to external source |
| V-P04 Business rules complete | ✅ | 8 rules documented |
| V-P05 External macros flagged | ⚠️ | %load_lookup source file unconfirmed |
| V-P06 %INCLUDE noted | — | No %INCLUDE in this script |
| V-P07 Cross-script macro vars | 🔧✅ | &max_date cross-script dep added |
| V-P08 Hard-coded values | ✅ | 2 hard-coded values flagged (F01, F02) |
````

---

## 15. Doubt Escalation Protocol

When the agent encounters ambiguity, conflicting evidence, or missing information that cannot
be resolved from the script alone, it must **stop and ask the user** rather than proceeding
with an assumption that could silently propagate an error through the entire knowledge base.

**The rule**: If proceeding requires an assumption about intent, data content, system behavior,
or external context — and that assumption would affect lineage, business rules, or join logic —
escalate. Do not guess.

---

### 15.1 When to Escalate

Escalate immediately when:

- A dataset is referenced but its source cannot be confirmed from LIBNAMEs in scope.
- A column appears in output but its derivation is not visible in the step code.
- A MERGE has no `IF` condition and the intended join type is not clear from context.
- A macro variable is used but its assignment is not found in any analyzed script.
- A `CASE WHEN ... ELSE` or `SELECT ... OTHERWISE` has an `OTHERWISE` or `ELSE` that
  may be catching unexpected values — and the intent is unclear.
- A `%INCLUDE` references a file not in the analyzed script set.
- Two interpretations of the same code are equally plausible.
- A calculated value doesn't match what the business context would suggest.
- An execution order is ambiguous and the correct order changes which macro variables or
  datasets are available at each point.
- A validation check from Section 14 cannot be resolved from the script alone.

Do NOT escalate for:
- Clear, unambiguous SAS syntax with standard behavior.
- Anti-patterns that are simply flagged (not requiring user decision to proceed).
- Minor style issues that don't affect lineage or correctness.

---

### 15.2 Doubt Record Template

When escalating, produce a Doubt Record using this exact template. Every field is mandatory.
Do not proceed with analysis of the affected element until the user responds.

````markdown
---
## ❓ DOUBT RECORD

**Doubt ID**: D-<NNN>
  *(Sequential number within this session. E.g., D-001, D-002. Reset per session.)*

**Raised At**: <timestamp or session step>

**Doubt Type**: <code from Section 15.3>

---

### 📄 Location

**Script File**: `<filename>.sas`
**Script MD File**: `scripts/script_<NNN>_<name>.md`
**Step Number**: Step <N>
**Step Type**: <DATA / PROC SQL / PROC SORT / MACRO / etc.>
**Line Numbers**: Lines <start> to <end>
  *(Provide the exact line range from the .sas source file where the ambiguity appears.)*

---

### 💬 Code in Question

```sas
<paste the exact lines of code that are ambiguous>
```

---

### ❓ The Doubt

**What is unclear**:
<1–3 sentences stating precisely what is ambiguous or unknown. Be specific — name the
exact variable, table, column, condition, or behavior that cannot be determined.>

**Why it matters**:
<1–2 sentences explaining what downstream analysis would be affected if this is resolved
incorrectly. E.g., "If this is an INNER JOIN, 40% of transaction rows may be dropped
from lineage tracing. If it is a LEFT JOIN, all rows are retained.">

**What has already been checked**:
<List the things already examined in an attempt to resolve this without escalation. E.g.,
"Checked all LIBNAME declarations — no mapping for REFLIB found. Searched all preceding
steps — REFLIB.product_master is not produced by any prior step. Searched %INCLUDE files
listed — none match.">

---

### 🔀 Options Being Considered

**Option 1**: <label>
<Description of what this interpretation means for the analysis.>
*Implication*: <What the lineage / business rule / join output would look like under this option.>
*Evidence for*: <Any code signals or context that support this interpretation.>
*Risk if wrong*: <What breaks if this is chosen incorrectly.>

**Option 2**: <label>
<Description.>
*Implication*: <...>
*Evidence for*: <...>
*Risk if wrong*: <...>

*(Add Option 3+ if applicable. Maximum 4 options — if more than 4 exist,
the doubt is under-specified. Narrow it further before escalating.)*

---

### 🧠 Agent's Leaning

**Preferred option**: Option <N> — <label>
**Confidence**: <Low / Medium / High>
**Reasoning**: <1–3 sentences explaining why this option appears most likely, citing
specific code evidence, SAS behavior rules, or business context clues.>

---

### 💡 What the Agent Needs From the User

<A specific, answerable question. Not "what should I do?" but a concrete ask. Examples:>
- "Can you confirm whether REFLIB maps to the Oracle finance schema or the SAS permanent library
  at D:\data\ref\?"
- "Should the MERGE here be treated as an inner join (IF a AND b) or is the missing IF condition
  intentional (full outer join)?"
- "Is the &adj_factor variable set by a prior job step or controller script before this script runs?"
- "Are there values of status_code other than 'A', 'I', 'S' in the source data that the
  OTHERWISE clause needs to handle?"

**Do you have a better interpretation or additional context not visible in the code?**
*(Free text — the user may have business knowledge or system documentation that resolves this
  without choosing from the options above.)*

---

### ⏸️ Analysis Paused

The following analysis is paused pending resolution of this doubt:
- <Step N — specific element that cannot proceed>
- <Any downstream steps that depend on this element>

Analysis of unaffected steps will continue. This doubt will be logged in
`/kb/doubt_log.md` and the Validation Summary for this script will show `❓` for
the relevant check until resolved.

---
````

---

### 15.3 Doubt Type Codes

| Code | Doubt Category | Typical Trigger |
|------|---------------|----------------|
| D-01 | Column origin unknown | Column in output has no visible source in step code |
| D-02 | Dataset source unresolved | Dataset referenced but no LIBNAME or producing step found |
| D-03 | Join type ambiguous | MERGE without IF condition; JOIN conditions unclear |
| D-04 | Pre-sort missing or unverified | MERGE without confirmed prior PROC SORT |
| D-05 | RETAIN reset logic unclear | RETAIN with no visible FIRST. reset; reset logic is conditional |
| D-06 | Macro variable unresolved | `&var` used but assignment not found in any analyzed script |
| D-07 | NODUPKEY vs NODUP intent | Both present; or NODUP where NODUPKEY is likely intended |
| D-08 | Character length risk | Length inferred from short first assignment; longer values possible |
| D-09 | CALCULATED chain broken | `CALCULATED col` where `col` is not defined in the same SELECT |
| D-10 | NULL IN / NOT IN risk | Subquery in IN/NOT IN may return NULL — behavior unclear |
| D-11 | Step count mismatch | Script step count doesn't match MD step inventory count |
| D-12 | Lineage chain incomplete | Chain terminates at intermediate with no further trace |
| D-13 | Business rule intent unclear | Filter or condition present but business meaning is ambiguous |
| D-14 | External macro/include unknown | Macro called from file not in analyzed script set |
| D-15 | Cross-script macro var timing | Macro variable produced/consumed order is ambiguous |
| D-16 | Hard-coded value intent | Literal value — unclear if business parameter or structural constant |
| D-17 | Execution order ambiguous | Cannot determine which script runs before which |
| D-18 | Dead output suspected | Permanent output table never consumed — may be dead code |
| D-19 | Business logic contradiction | Two rules or conditions appear to contradict each other |
| D-20 | Data type mismatch | Column type in JOIN key or comparison appears to differ between sides |
| D-21 | Format/informat mismatch | Date or numeric format applied appears inconsistent with data content |
| D-99 | Other | Any ambiguity not covered by D-01 to D-21 |

---

### 15.4 Doubt Log File

All Doubt Records are persisted to `/kb/doubt_log.md`. This file:
- Contains every Doubt Record raised across all sessions, in chronological order.
- Records the user's response and the resolution applied.
- Is read at the start of every new session to understand what was previously uncertain
  and how it was resolved.

Format for the resolution section (appended to each Doubt Record after user responds):

````markdown
### ✅ Resolution

**Resolved**: <date>
**Resolution Type**: <User clarified / Agent re-examined / Assumption accepted / Deferred>
**User Response**: <Exact or paraphrased user response>
**Resolution Applied**:
<What was changed in the analysis, MD file, or master_index as a result.>
**Principle Derived**:
<Optional: a general rule that can be applied to similar situations in future sessions.
 E.g., "In this system, all REFLIB references point to D:\data\reference\ — confirmed by user.">
**Skill Learnings Update**: <Yes — added to skill_learnings.md entry for this session / No>
````

---

### 15.5 Partial Continuation Rule

When a doubt is raised and analysis is paused for the affected element, the agent must
continue analyzing all other elements that are NOT dependent on the unresolved doubt.

**Dependency rule for partial continuation**:
- A step is blocked if it reads from a table or uses a column/macro variable whose resolution
  is the subject of the open doubt.
- A step is NOT blocked if it uses completely independent tables, columns, and macro variables.
- The Validation Summary entry for any blocked item shows `❓` with the Doubt ID referenced.

Example:
```
Doubt D-001 raised: source of REFLIB.product_master unclear.

Blocked:
  - Step 05 attribute derivation (uses REFLIB.product_master.category)
  - Lineage for product_category in Attribute Dictionary
  - Business Rule R004 (filter on product_category)

Not blocked:
  - Step 03 (reads SRC.raw_txn — unrelated to REFLIB)
  - Step 04 (PROC SORT on WORK.filtered — unrelated)
  - Step 06 (PROC MEANS — depends on Step 05 output, but only on amount columns,
             not on product_category)
```

Document the blocked and not-blocked list in every Doubt Record under "Analysis Paused".

---

### 15.6 Escalation Tone and Format

When presenting a Doubt Record to the user:
- Lead with a one-sentence plain-language summary of the problem before the full template.
- Keep technical detail in the template, not in surrounding prose.
- Never present more than 2 Doubt Records at once. If multiple doubts exist, prioritize by
  dependency — resolve blockers first.
- After the user responds, confirm the resolution explicitly before continuing:
  *"Understood — I'll treat this MERGE as an inner join. Continuing analysis of Step 05."*
- If the user's response resolves the doubt only partially, raise a follow-up Doubt Record
  referencing the original: `D-<NNN>-B`.

---

*SAS Engineer Skill — sas_engineer/SKILL.md*
*Validation: run Section 14 checks at every step, script, and index boundary.*
*Doubt escalation: use Section 15 template whenever proceeding requires an unverified assumption.*
*Self-refinement: update /kb/skill_learnings.md after every session.*
