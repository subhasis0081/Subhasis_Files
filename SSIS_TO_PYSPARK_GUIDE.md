# SSIS (.dtsx) → PySpark (Microsoft Fabric) — Conversion Spec

**Role:** Senior SSIS→PySpark migration engineer. Input: `.dtsx` (+ optional AI draft). Output: one Fabric-notebook `.py` that is **runnable and output-identical to SSIS**.
**Priority order:** 1) same data as SSIS  2) runs in Fabric  3) readable. Never "improve" business logic.

---

## 0. Workflow (follow in order)

1. **Parse the dtsx first.** Extract: variables (+ default values), connection managers, control-flow order (`PrecedenceConstraints`), every task's SQL / expressions, every data-flow component (`SqlCommand`/`OpenRowset`, `AccessMode`, Derived Column expressions, Lookup SQL + `NoMatchBehavior`, Conditional Split conditions, destination **Input Columns / ExternalColumns**).
2. **Draw the execution graph** from `Package.PrecedenceConstraints` (and per-container ones). Code order = graph order. A table/temp must exist before it is read.
3. **Write code** per the rules below, one commented block per SSIS component.
4. **Verify** (§9). Report assumptions/flags (§10).
5. If the dtsx is ambiguous → pick the **safest** assumption, comment `# FLAG:` and list it. Never invent business rules.

---

## 1. File skeleton (always this order)

```python
from datetime import datetime, date, timedelta
from delta.tables import DeltaTable
from pyspark.sql import functions as F
from pyspark.sql.functions import col          # only if bare col() is really used
from pyspark.sql.window import Window          # only if used

# --- Parameters (TODO: set actual Fabric workspace names before running) ---
EDW_Workspace = "EDW"
RegulatoryReports_Workspace = "RegulatoryReports"

EDW_ABFS = f"abfss://{EDW_Workspace}@onelake.dfs.fabric.microsoft.com/EDW.Lakehouse/Tables/"
RegulatoryReports_ABFS = f"abfss://{RegulatoryReports_Workspace}@onelake.dfs.fabric.microsoft.com/RegulatoryReports.Lakehouse/Tables/"

# --- Source tables: ONLY tables that are actually READ in the package ---
CTRL_BATCH_DATE = spark.read.format("delta").load(f"{EDW_ABFS}ctrl/BATCH_DATE")

# --- Variables (SSIS defaults) ---
cvarBatchDate = "0000-00-00"
PackageName = "<EXACT_PACKAGE_NAME>"      # literal string, NOT "System::PackageName"

# ... components in control-flow order ...

r"""
Script Architecture:
Package\<task 1>
Package\<task 2>
"""
```

### ABFS path variables — one per SSIS connection manager
Derive one variable per **source/target database (connection manager)** used in the package. Pattern:

`<Name>_Workspace = "<Name>"` and `<Name>_ABFS = f"abfss://{<Name>_Workspace}@onelake.dfs.fabric.microsoft.com/<Name>.Lakehouse/Tables/"`

| SSIS connection / DB | Variables |
|---|---|
| EDW | `EDW_Workspace`, `EDW_ABFS` |
| EDW_UAT | `EDW_UAT_Workspace`, `EDW_UAT_ABFS` |
| RegulatoryReports | `RegulatoryReports_Workspace`, `RegulatoryReports_ABFS` |
| EDO | `EDO_Workspace`, `EDO_ABFS` |
| EDO_Reports | `EDO_Reports_Workspace`, `EDO_Reports_ABFS` |
| any other `<X>` | `<X>_Workspace`, `<X>_ABFS` |

- Name = the connection manager / database name from the dtsx, exactly. Declare **only** the ones the package uses.
- Every read, write, `DeltaTable.forPath`, and merge uses these variables — never a hardcoded path.
- Never use the old `*_Lakehouse` name.

Rules
- Naming: `SCHEMA_TABLE` upper-case (`CTRL_BATCH_DATE`, `N4738_SSIS_LOG`). Path = `{<Name>_ABFS}schema/TABLE`.
- **No unnecessary functions.** Write straight-line, top-to-bottom code that mirrors the SSIS flow. Define a helper only when the same non-trivial logic is genuinely repeated (3+ places) or SSIS itself repeats a block (e.g. a shared export). No wrappers around single calls, no helper per component, no classes. If a helper exists, keep it small and place it right under the imports.
- `spark` is the Fabric-injected session — never create one.
- No `dbutils`. Files via `mssparkutils` / `Files/...` paths only.
- Do **not** load write-only (truncate+reload) targets at the top.
- Trailing architecture docstring is **raw** (`r"""`) — paths contain `\U`, `\E`.
- Drop unused imports. Never `from pyspark.sql.functions import sum/max/min/filter/first` (shadows builtins) — use `F.`.
- Reusable helper allowed for date normalisation (`_to_date(v, default)` handling datetime/date/str).

---

## 2. Component block format (mandatory)

```python
# Following is the Microsoft.<ComponentClass> Component, with refId Package\<Container>\<Task>
# T-SQL: <one-line summary or the exact predicate that matters>
<DataFrame or action>
```
Variable name = readable form of the component name (`DFT_N4738_VAT_X_CST_Derived_Column`). Never start an identifier with a digit (`7_SET_X` → `SET_X`).

---

## 3. Component → PySpark map

| SSIS | PySpark |
|---|---|
| Execute SQL (SELECT → variable) | `.agg/.select(...).first()` → Python var |
| Execute SQL (`#temp` build) | DataFrame chain; temp **stays a DataFrame** |
| OLE DB Source (table) | `spark.read.format("delta")` + explicit `.select` of dtsx columns |
| OLE DB Source (`OpenRowsetVariable`) | resolve the variable's value (`#tmp` → the DataFrame built earlier) |
| Derived Column | `withColumn` (**new** column ≠ existing; check the dtsx `name=`) |
| Conditional Split | sequential `filter`s; evaluate in order; default output = remainder |
| Lookup, only Match output wired | **inner** join |
| Lookup, No-Match wired / redirected | `left_anti` for no-match, inner for match |
| Lookup, Ignore failure | **left** join (+ default values) |
| Merge Join / Union All / Multicast | `join(how)` / `unionByName` / plain reuse of the DF |
| Sort + Row Number | `Window.partitionBy().orderBy()` + `row_number()` — **always** an `orderBy` |
| Aggregate | `groupBy().agg(... .alias())`; GROUP BY w/o aggregate → `.distinct()` |
| Data Conversion | `.cast(...)` matching SSIS type |
| Flat File Destination | `coalesce(1).write.csv` with delimiter/qualifier/encoding from the connection manager |
| Script Task (CA7 / file post-processing) | **skip** with comment + FLAG |
| For Loop | `while` over dates (Init/Eval/Assign expressions copied in comments) |
| OnError handler | not replicated; FLAG |

---

## 4. Write / DML conventions (agreed — apply everywhere)

| SSIS | PySpark |
|---|---|
| `TRUNCATE` + full reload of **same** table | **one** `mode("overwrite")` at the final destination. No empty-DF hacks, no `StructType([])`, no `saveAsTable` truncates |
| `DELETE ... WHERE` (partial) | `DeltaTable.forPath(spark, path).delete(<cond>)` |
| `DELETE A FROM A JOIN B` | `DeltaTable.merge(...).whenMatchedDelete()` (dedupe source on keys) |
| `UPDATE` on a real table (post-load) | persist first, then `DeltaTable.update` / `.merge` |
| `UPDATE` on `#temp` | DataFrame `withColumn` |
| plain `INSERT` | `.mode("append")` |
| Upsert | `DeltaTable.merge(...).whenMatchedUpdate(set=...)` |
| DELETE-then-INSERT slice reload | `DeltaTable.delete(batch cond)` then `append` |

### Physical-table DML — mandatory rules
- **DELETE on a physical (lakehouse) table → always `DeltaTable.forPath(spark, path).delete(<condition>)`.**
  Never emulate it with `filter` / anti-join / `dropDuplicates` followed by an overwrite.
  ```python
  DeltaTable.forPath(spark, f"{RegulatoryReports_ABFS}gef3/GETAPPLICATIONS").delete(
      (F.col("REQUESTDATE") <= F.lit("2026-03-09")) & (F.col("DT_REC") == F.lit("2026-03-12")))
  ```
  `DELETE A FROM A JOIN B` → `DeltaTable.merge(...).whenMatchedDelete()`.
- **UPDATE on a physical table (after it is loaded) → apply `DeltaTable.update(condition=..., set={...})` directly on the lakehouse table.**
  Never rebuild the table with `withColumn` / `when().otherwise()` and write it back. For join-based updates use `DeltaTable.merge(...).whenMatchedUpdate(set=...)`.
  ```python
  DeltaTable.forPath(spark, f"{EDW_ABFS}dbo/TBL").update(
      condition=F.col("FLAG") == 0, set={"FLAG": F.lit(1)})
  ```
  Order: write/load the table first, then run `.update` / `.merge` (as SSIS does).
- Only `#temp` tables (in-memory DataFrames) use `withColumn` for updates.
- Exception: TRUNCATE + full reload of the same table → single `mode("overwrite")` write.

Never silently change write mode, join type, filter, grain, or row-number order.

---

## 5. SSIS_LOG / control pattern (copy exactly)

```python
# START — append, ROW_ID = max+1 (IDENTITY emulation), aligned to full schema
StartTime_var = datetime.now()
_max = LOG.agg(F.max("ROW_ID").alias("m")).collect()[0]["m"]
pckg_run_ID = (int(_max) if _max is not None else 0) + 1
_row = spark.createDataFrame([(pckg_run_ID, PackageName, StartTime_var, rvarBatchDate)],
                             ["ROW_ID", "PackageName", "StartTime", "rvarBatchDate"])
_row.select([(F.col(f.name).cast(f.dataType) if f.name in _row.columns
              else F.lit(None).cast(f.dataType)).alias(f.name) for f in LOG.schema.fields]) \
    .write.format("delta").mode("append").save(f"{<Name>_ABFS}<schema>/SSIS_LOG")

# ... package work ...

# END — LAST statement of the script
DeltaTable.forPath(spark, f"{<Name>_ABFS}<schema>/SSIS_LOG").update(
    condition=F.col("ROW_ID") == F.lit(pckg_run_ID),
    set={"EndTime": F.current_timestamp(), "Final_Status": F.lit("Completed")})
```
Batch date: `CASE WHEN cvarBatchDate='0000-00-00' THEN date_format(CurrentBatchDate,'yyyy-MM-dd') ELSE cvarBatchDate END` (same for Next/Previous).
Use the log the dtsx actually writes (`gef3.SSIS_LOG` vs `n4738.SSIS_LOG` vs `dbo.TABLE_LOAD_LOG`) — never touch a log the package doesn't touch.

---

## 6. Expression & SQL translation cheatsheet

| SSIS / T-SQL | PySpark |
|---|---|
| `REPLACENULL(x,"")` / `ISNULL(x,0)` | `F.coalesce(x, F.lit(...))` |
| `(DT_DBDATE)x` | `F.to_date(x)` |
| `GETDATE()` | `F.current_timestamp()` (timestamp, **not** `current_date`) |
| `LEN` / `LTRIM RTRIM` | `F.length` / `F.trim` |
| `SUBSTRING(s,1,n)` | `F.substring(s,1,n)` (1-based, same as SSIS) |
| `IIF(c,a,b)` / `CASE` | `F.when(c,a).otherwise(b)` — keep branch order |
| `x IN (..)` / `NOT IN (..)` | `isin` / `left_anti` (flag NOT-IN-NULL caveat) |
| `EOMONTH`, `DATEADD`, `DATEDIFF` | calendar table or `last_day`, `date_add`, `datediff`; flag approximations |
| `STRING_SPLIT` | `split` + `explode` |
| `FOR XML PATH` concat | `concat_ws(sep, sort_array(collect_set()))` (deterministic) |
| `UNION` | `unionByName(...).distinct()` · `UNION ALL` → `unionByName` only |
| `SELECT DISTINCT` | `.dropDuplicates()` / `.distinct()` — **only** where SQL has it |
| `ROW_NUMBER() OVER` | `row_number().over(Window...)` identical partition/order |
| correlated scalar subquery `(SELECT MIN(d) FROM cal WHERE d > X.date)` | compute once per distinct `X.date` (join + group-min), join back. **Never** `F.expr("(SELECT .. FROM <df alias>)")` |
| `SELECT ROW_ID WHERE ts=@ts` after insert | `max+1` (§5) |
| literal dates inside SQL (`'2026-03-09'`) | keep literal; FLAG |
| Greek / Unicode literals | keep exactly, UTF-8 |

---

## 7. Hard coding rules (the bugs that kept recurring)

1. **Operator precedence.** Parenthesise every comparison: `(a == b) & (c == d)`. Never `a == b & c == d`. Use `& | ~`, never `and/or/not`.
2. **Valid join types only:** `inner, left, left_outer, right, full, left_anti, left_semi` (`"left anti-join"` is invalid).
3. **Ambiguity:** alias every DataFrame in joins (`src`, `lkp`, `PER`); qualify `F.col("PER.ACC_ID")`; explicit `.select` after every join.
4. **No bare column names** (`CALC_BATCH_DATE`) outside `F.col()`/`F.lit()`.
5. **New derived column ≠ cast of itself.** If the dtsx output column is new, don't read it as input.
6. **Parenthesis balance.** A stray `)` before `how=` or a missing `))` closing `.filter(` / `.unionByName(` were the classic failures — `ast.parse` after every file.
7. **No `collect()` + `isin(list)`** for anti-filters → `left_anti`.
8. **No `dropDuplicates()`** unless SSIS/SQL has DISTINCT / UNION / remove-duplicates.
9. **No `broadcast`** unless the lookup is provably small.
10. **No hardcoded paths** like `/mnt/lakehouse/...`; use `{<Name>_ABFS}schema/TABLE` or `Files/...`.
11. **No `base_path` / `write_mode` placeholders** — every write has an explicit path and mode.
12. Write column names exactly as in the dtsx/DDL (Spark is case-insensitive by default, but keep fidelity).
13. Run DML (delete/update) **before** the action that consumes the table; Delta re-scans lazily.
14. Variables: `F.lit(rvarBatchDate)`, never `F.col(rvarBatchDate)`.
15. Never generate a write to a table the dtsx doesn't write.

---

## 8. Metadata drift & destination mapping

- Destination truth = **Input Columns → ExternalColumns**. Select/alias into exactly those target names.
- **Unmapped destination column** → inserted NULL (`F.lit(None).cast(type)`), even if upstream computed a value.
- Source `OpenRowsetVariable` whose cached column names differ from the `#temp` DDL/INSERT list: map by **business meaning**, comment `# FLAG:` and list it.
- Types from the `#temp` DDL (`int`, `nvarchar`, `datetime`) → cast accordingly.
- A Lookup/Multicast output with no path in `<paths>` is dead — don't build it.

---

## 9. Verification checklist (run before delivering)

```bash
python3 -I -c "import ast,sys; ast.parse(open(sys.argv[1]).read())" file.py
python3 -m py_compile file.py
python3 -m pyflakes file.py        # only "undefined name 'spark'" allowed
python3 -c "print(open('file.py','rb').read().count(b'\x00'))"   # must be 0
```
- [ ] Block order = dtsx precedence graph
- [ ] Every SQL task / data-flow component has a block (or an explicit "skipped + why")
- [ ] Join type, filter, DISTINCT, row-number order match the SQL
- [ ] Each target: correct write mode (§4) and path
- [ ] Log start (append) at top, log end (update) as the **last** statement
- [ ] No unused imports, dead variables, digit-leading names
- [ ] No unnecessary helper functions; straight-line code
- [ ] Every physical DELETE uses `.delete`, every physical UPDATE uses `.update`/`.merge` (no filter/withColumn rebuilds)
- [ ] One `<Name>_ABFS` per connection manager used; no hardcoded paths
- [ ] File tail intact (docstring closed). On this mount, files can be silently truncated or NUL-padded after edits — re-check size, NUL count and `ast.parse` after every write; write large files via bash heredoc.

---

## 10. Response format (when delivering)

1. **Problem summary** (2–3 lines)
2. **Corrected code** (file)
3. **Key fixes** (only meaningful ones)
4. **SSIS equivalence notes**
5. **Risks / validation points** — each `# FLAG:` item: assumption, why, how to verify on production data

Always check for: hardcoded literals in SQL · metadata drift · skipped Script Tasks · OnError logging not replicated · NOT IN vs `left_anti` NULLs · DATEDIFF/EOMONTH approximations · encoding change (cp1253 → UTF-8) · unknown PK forcing overwrite-with-passthrough · initial-branch filenames using stale loop variables.

---

## 11. Prompt to paste with a new package

> Convert the attached `.dtsx` to a Fabric PySpark notebook following `SSIS_TO_PYSPARK_GUIDE.md` exactly. Read the dtsx first (variables, precedence graph, SQL, data-flow components, destination mappings). Output a single `.py`, run the §9 checks, and answer in the §10 format. Do not simplify logic; flag every assumption.
