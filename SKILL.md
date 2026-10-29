---
name: supabase-medallion-powerbi
description: >
  Build an end-to-end analytics pipeline from raw CSV files to a Power BI report: a bronze/silver/gold
  (medallion) data warehouse in Supabase Postgres, a star schema in the gold layer, a read-only Power BI
  login with working RLS, and a Power BI Project (PBIP) with a TMDL semantic model, date table and measure
  table. It runs a structured decision interview with the user, backed by queries on their own data,
  so the modelling choices that change headline numbers (historical costs, missing dates, source
  conflicts, incomplete periods, non-additive counts, repo exposure) are made deliberately and logged.
  Use this skill whenever the user wants to load CSV or Excel exports into Supabase or Postgres and
  report on them in Power BI, mentions bronze/silver/gold or medallion architecture, star schema, fact and
  dimension tables, PBIP, TMDL, connecting Power BI to Supabase, or is stuck on Supabase-to-Power BI errors
  (certificate errors, empty tables, pooler login, TMDL load errors) — even if they don't say "medallion".
compatibility: >
  Supabase project (SQL Editor, Table Editor or the Supabase MCP server), Power BI Desktop on Windows,
  optional psql. Works with any CSV source; examples use a CRM/ERP sales dataset.
---

# Supabase medallion → Power BI (PBIP/TMDL)

This skill turns a folder of CSV files into a tested data warehouse in Supabase and a Power BI project
that reads from it. It encodes the order of work and the pitfalls that cost the most time in practice,
so follow the phases in order and run the checks at the end of each one. Every phase leaves something
verifiable (row counts, zero-error checks, a refreshed model), which is how you avoid debugging three
layers at once later.

## Working with the user: the decision interview

The mechanics below are the same for every dataset; the decisions are not. Whether margin uses
today's cost or the cost at the time of sale, whether a missing date may be derived, which system wins
when two sources disagree – these change headline numbers, and only the user knows what the numbers
are for. So the skill interviews the user at four points, using evidence from their own data:

| Round | When | Typical decisions |
|---|---|---|
| 1 | Before building | Purpose and audience, grain, business keys, dimension history, refresh cadence |
| 2 | After silver | Historical prices on facts, missing dates, source conflicts, unknown members, duplicates |
| 3 | After gold, before the report | Non-additive counts, incomplete edge periods, "current month", units, unsupported KPIs |
| 4 | Before hand-over | Repo visibility and identifiers, passwords and access, documentation |

**Read `references/decision-interview.md` before Round 1.** It holds the protocol and a question bank
with detection SQL. The short version:

- Investigate first, then ask – never ask what a query can answer; skip questions whose detection finds
  nothing (log them as "not applicable – checked").
- **One question per message**, with the evidence in numbers, 2–4 lettered options, and your
  recommendation and why. Wait for the answer before the next question.
- Log every decision in `docs/model-decisions.md` and implement each round's decisions together, then
  rerun the quality checks.
- If the user says "use your recommendations for the rest", do so, but still log each decision and
  list them in the hand-over.

## Phase 0 – Understand the sources and agree the model

Before writing any SQL, profile every CSV (header, row count, sample rows, obvious data-quality issues),
then run **interview Round 1**. These answers drive every later decision:

- **Purpose and audience** – demo/reference, reusable template or production. It decides how much
  automation, history and hardening is worth building.
- **Grain of each fact table** – e.g. "one row per order line (order number + product)".
- **Business keys** and how sources join (watch for key formats that differ between systems, such as
  `AW-00011000` vs `AW00011000` vs `NASAW00011000`; silver must normalise them).
- **Facts vs dimensions**, and **cardinality** of every relationship (should be many-to-one from fact).
- **History**: are dimension rows versioned (start/end dates)? Decide the policy now; values that change
  over time (cost, price) are handled in Round 2.

Propose the star schema (tables, columns, PK/FK) and get approval. A DBML diagram for
https://dbdiagram.io is a quick way to show it.

## Phase 1 – Schemas and bronze

1. Create `bronze`, `silver`, `gold` schemas. Ship every DDL change as a numbered migration file
   (`supabase/migrations/<timestamp>_<name>.sql`) so the database can be rebuilt; use
   `create ... if not exists` so reruns are safe.
2. One bronze table per CSV, **columns in the same order as the CSV header, all lower case**, types as
   close to the source as possible (integers stay integers, dates stored `yyyymmdd` stay integers – they
   are parsed in silver). See `references/sql-templates.md` → *Bronze*.
3. Load the data, then compare row counts with the files (`lines - 1`, or `lines` if the last line has
   no trailing newline). Options, best first:
   - `psql \copy` through the **session pooler** (fast, scriptable). Template in the SQL reference.
   - Supabase **Table Editor → Insert → Import data from CSV**. It matches columns by exact header name,
     so **upper-case headers silently fail** – lower-case the header line first. It aborts the whole file
     on one bad value, so read the error banner.
   - For tiny lookup files only: `insert ... values` through SQL/MCP.

Do not clean data in bronze. Bronze is the replayable copy of the source; cleaning there destroys the
evidence you need when a number looks wrong later.

## Phase 2 – Silver (clean and standardise)

Write one procedure, `silver.load_silver()`, that truncates and reloads every silver table from bronze.
Typical transformations (patterns in `references/sql-templates.md` → *Silver*):

- `trim()` text, map codes to readable values (`'M' → 'Male'`, `'S' → 'Single'`, unknown → `'n/a'`).
- Keep the latest row per business key with `row_number() over (partition by key order by created desc)`.
- Parse integer dates (`20131229`) to `date`, turning `0`, wrong-length or out-of-range values into NULL.
- Recompute derived values that are inconsistent (e.g. `sales <> quantity * price`).
- Normalise country/gender variants (`DE`→`Germany`, `US`/`USA`→`United States`, blank → `'n/a'`).
- Null out impossible values (future birth dates) rather than deleting rows.

Give the procedure `set search_path = ''` and schema-qualify every table name. Supabase's security
advisor flags procedures with a mutable search path, and the fixed path removes a class of
search-path hijacking.

When silver loads, run **interview Round 2**: check for versioned prices/costs, missing or invalid
dates, conflicting values between sources, unknown members and duplicates, and put each finding to the
user. The fixes (point-in-time values, derived dates with a flag, master-source rules) land in silver
and gold, so decide them before building gold.

## Phase 3 – Gold (star schema)

Build the dimensions and facts as **tables filled by `gold.load_gold()`**, not as views. Views
recompute surrogate keys on every query, so keys can shift if silver reloads during a Power BI refresh,
and Power BI imports from tables are faster and predictable.

- Surrogate keys `<name>_key` generated with `row_number()` in the load; keep business keys as columns.
- Fact table stores the dimension keys plus measures; date columns stay real `date` columns.
- Add indexes on fact foreign keys and the main date column.

Template: `references/sql-templates.md` → *Gold*.

## Phase 4 – Quality checks

Run checks that return **0 on success** so a glance tells you the state (templates in the SQL reference):

- Row counts per layer vs expected.
- Duplicate / NULL keys in every dimension; orphaned fact rows (fact key not found in dimension).
- Business rules (sales = quantity × price, order date ≤ ship date, no negative cost).
- Distinct values of every standardised column (to spot a variant you missed).

Report known, accepted exceptions explicitly (e.g. "19 order lines have an invalid source date"), and
tell the user how they will show up in the report (in totals, missing from date-based visuals).

Then run **interview Round 3** on the gold data, before any visual is built: do breakdowns of distinct
counts add up to the total, are the first and last periods complete (and is the last one plausible,
not a truncated extract), what counts as the "current" month, which units to show, and which requested
KPIs the data cannot support.

## Phase 5 – Security for Power BI

Power BI should log in with its own read-only role, never with `postgres`. Template:
`references/sql-templates.md` → *Security*.

- `create role powerbi_reader login` with `default_transaction_read_only = on`, `usage` on `gold`,
  `select` on its tables and `alter default privileges` so new gold tables are readable too.
- **If RLS is enabled on the gold tables (Supabase often enables it), add a `select` policy for the
  role.** RLS with no policy is default-deny: the login succeeds and every table returns **0 rows**,
  which looks like a broken refresh rather than a permission problem.
- Let the user set the password themselves (`alter role powerbi_reader with password '...'`); never
  write it into a file, migration or chat.
- Run the Supabase security advisor afterwards; aim for zero findings.

## Phase 6 – Connect Power BI

Walk the user through these, in this order:

1. **Host**: Supabase → *Connect* → **Session pooler** host and port 5432 (works on IPv4 networks; the
   direct host is often IPv6-only).
2. **User name format for the pooler**: `powerbi_reader.<project-ref>` – without the project ref the
   pooler rejects the login. Database: `postgres`. Credential type: *Database*, not Windows.
3. **Certificate**: Power BI's PostgreSQL connector validates TLS, and Supabase uses its own root CA, so
   the first connection fails with *"The remote certificate is invalid according to the validation
   procedure"*. Fix: Supabase → *Project Settings → Database → SSL Configuration → Download
   certificate* → double-click → *Install Certificate* → *Current User* → *Trusted Root Certification
   Authorities*, then restart Power BI. Recommend this over unticking "Encrypt connections", which
   sends credentials unencrypted.

## Phase 7 – The PBIP project and TMDL model

A PBIP project is text, which makes it reviewable and diffable. Structure and templates:
`references/tmdl-templates.md`.

```
<Name>.pbip                      # entry file – must sit next to the two folders below
<Name>.Report/                   # report (PBIR)
<Name>.SemanticModel/definition/ # model (TMDL): model, expressions, relationships, tables/
```

- Put the server and database in **Power Query parameters** and one shared source query, so
  switching environments is a parameter change.
- One import query per gold table: select columns, set types.
- Generate `dim_date` in Power Query from the fact's dates (whole years), mark it as the date table
  (`dataCategory: Time`, `isKey` on the date column) and relate it to the main fact date. Add extra fact
  dates (ship, due) as **inactive** relationships and expose them through `USERELATIONSHIP` measures.
- Keep all measures in one measure table (`_Measures`) with display folders. Use `DIVIDE`, variables,
  and format strings. If the data ends mid-month, compare the **latest complete month** with the one
  before it – otherwise every "this month vs last month" card shows a misleading drop.
- Hand the user a short test: expected totals from SQL vs the values in a card or table visual.

### TMDL editing rules (these are the ones that break loading)

- **Power BI Desktop must be closed** while you edit the files. Desktop does not reload them and will
  overwrite your edits on its next save. A windowless `PBIDesktop.exe` can linger after closing; it is
  harmless if it has no window, but a reboot clears it.
- Indent with **tabs**. Quote names with spaces or special characters: `measure 'Total Orders' = ...`.
  Multi-line DAX goes one indentation level deeper than the object's properties.
- `///` descriptions are valid on tables, columns and measures but **not on relationships** – Desktop
  fails with *"Property 'description' is unknown"*.
- Every object needs a unique `lineageTag` (GUID). Generate them; never copy them between objects.
- A measure-only table needs a calculated partition and one hidden column (template provided).

## Phase 8 – The report

- The report needs a **base theme file** and **at least one page**. Without a theme Desktop crashes on
  open with `Cannot read properties of undefined (reading 'customTheme')`. Copy the base theme JSON
  from any project saved by a current Power BI Desktop
  (`StaticResources/SharedResources/BaseThemes/<theme>.json`) and reference it from the report.
- Desktop may convert `report.json` to the PBIR folder format (`definition/pages/...`) and rename the
  entry file (`<Name>1.pbip`) on save. Both are normal; rename the entry file back while Desktop is closed.
- Build visuals with a PBIR-aware tool if one is available (for example the `powerbi-report-author` CLI
  for validation and previews). Validate after every batch of edits and review a screenshot of every
  changed page before calling it done – schema-valid PBIR can still render badly.
- New **card visuals (`cardVisual`) draw their own inner tile** (`fillCustom`, white by default) on top
  of the container background. Dark KPI tiles therefore render as white boxes with a thin dark frame and
  white labels disappear. Set `fillCustom.transparency` to 100 (selector `id: default`).

## Phase 9 – Hand-over

Run **interview Round 4** first: is the repository public, and do committed files contain the project
ref, pooler host or `<role>.<project-ref>` user names; who holds the password; who reads the guide.
Then finish with:

1. Expected row counts and key totals (so the user can verify a refresh), and the decision log.
2. Refresh procedure: load new CSVs into bronze → `call silver.load_silver(); call gold.load_gold();`
   → Refresh in Power BI. Offer `pg_cron` for scheduled loads (template in the SQL reference).
3. Commit migrations, tests and the PBIP to Git; make sure `.pbi/localSettings.json` and
   `.pbi/cache.abf` are git-ignored and no password is in any file.
4. A step-by-step guide for the user's team if they want one.

## When something goes wrong

Read `references/troubleshooting.md`; it maps every error we have seen to its cause and fix (CSV
import, row counts, RLS, certificates, pooler login, PBIP/TMDL load errors, theme crashes, white KPI
tiles, full disks). Two general rules:

- When Desktop shows an error dialog with a stack trace, the innermost exception names the TMDL
  object type that failed to load (e.g. `SingleColumnRelationship`) – start there.
- Low free disk space (under a few hundred MB on the system drive) causes unrelated-looking failures
  in installers, Power BI and shells. Check it early and tell the user before deleting anything.

## Reference files

- `references/decision-interview.md` – interview protocol, decision-log template and question bank
  (with detection SQL) for all four rounds. Read before Round 1.
- `references/sql-templates.md` – bronze, load, silver, gold (incl. point-in-time values and derived
  dates), quality checks, security, pg_cron.
- `references/tmdl-templates.md` – PBIP layout, parameters, partitions, date table (incl. complete-month
  flag), measure table, measures, relationships, minimal report.
- `references/troubleshooting.md` – error → cause → fix.
