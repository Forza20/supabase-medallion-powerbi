# Troubleshooting: error → cause → fix

Grouped by phase. Each entry was hit in a real build.

## Loading CSV into Supabase

| Symptom | Cause | Fix |
|---|---|---|
| Table Editor import "succeeds" but the table is empty, or columns stay NULL | Headers do not match column names exactly (e.g. `CID` vs `cid`) | Lower-case the header line, then import again |
| Import aborts with a type error | One value does not fit the column type (text in an integer column, bad date) | Read the error banner; import as `text` in bronze and cast in silver |
| Row count is one higher/lower than `wc -l` | The file has no trailing newline (or has a blank last line) | Compare with `lines - 1` and `lines`; check the last line |
| `winget install PostgreSQL...` fails with "Installer hash does not match" | Stale package manifest | Install psql from the vendor's installer (command-line tools only), or use the Table Editor |
| `psql` login fails through the pooler | User name lacks the project ref | Use `postgres.<project-ref>` (or `<role>.<project-ref>`) |

## Silver / gold

| Symptom | Cause | Fix |
|---|---|---|
| Fewer silver rows than bronze | Dedupe keeps latest row per key (intended) or NULL keys were filtered | Report the difference and why |
| Orphan keys in the fact | Business-key formats differ between systems | Normalise keys in silver (`replace`, `substring`, prefixes) |
| Security advisor: "Function Search Path Mutable" | Procedure without fixed `search_path` | `alter procedure ... set search_path = ''` and schema-qualify names |
| Surrogate keys change between refreshes | Gold built as views with `row_number()` | Materialise gold as tables loaded by a procedure |

## Power BI connection

| Symptom | Cause | Fix |
|---|---|---|
| "The remote certificate is invalid according to the validation procedure" | Supabase root CA not trusted by Windows | Download the certificate (Project Settings → Database → SSL Configuration), install to *Current User → Trusted Root Certification Authorities*, restart Power BI |
| Login fails / "password authentication failed" | Pooler user name without project ref, or Windows credentials chosen | `powerbi_reader.<project-ref>`, credential type *Database* |
| Connection works but every table has 0 rows | RLS enabled with no policy for the role (default-deny) | `create policy ... for select to powerbi_reader using (true)` per table |
| Cannot reach the direct host | Direct host is IPv6-only on many plans | Use the session pooler host (port 5432) |

## Opening the PBIP

| Symptom | Cause | Fix |
|---|---|---|
| "ReportDefinition: Required artifact is missing in ...definition.pbir" | `.pbip` copied without its `.Report`/`.SemanticModel` folders (e.g. downloaded alone) | Open the `.pbip` inside the project folder; download the whole repo/zip |
| "Property 'description' is unknown and is not expected" (inner exception names `SingleColumnRelationship`) | `///` description in front of a relationship | Remove the `///` line; descriptions are not supported on relationships |
| Any TMDL load error after editing | Spaces instead of tabs, unquoted names, duplicate `lineageTag`, or Desktop was open during the edit | Check indentation and quoting; close Desktop before editing |
| Desktop crashes rendering: "Cannot read properties of undefined (reading 'customTheme')" | Report has no base theme | Add the theme file under `StaticResources/SharedResources/BaseThemes` and reference it in `report.json` |
| Desktop opens nothing / double-click does nothing | A hung `PBIDesktop.exe` without a window, or a nearly full disk | Reboot; free disk space |
| Entry file reappears as `<Name>1.pbip`, `report.json` turns into `definition/pages/...` | Desktop saved the project in its newer format | Normal. Rename the entry file back while Desktop is closed; commit the new format |
| Edits to TMDL/PBIR disappeared | Desktop was open and saved over them | Close Desktop (choose *Don't save*) before editing; re-apply edits |

## Report visuals

| Symptom | Cause | Fix |
|---|---|---|
| Dark KPI tiles render white with a thin dark frame; labels missing | `cardVisual` inner tile (`fillCustom`) drawn over the container background; white labels on white | `fillCustom.transparency = 100` with selector `id: default` |
| Small sparkline/area charts show an axis title (e.g. `month_start`) even with the axis hidden | `showAxisTitle` defaults to on | Set `showAxisTitle: false` on `categoryAxis` and `valueAxis` |
| Line chart shows an unwanted vertical slider | Zoom slider shows on both axes by default | `zoom.showOnValueAxis: false` |
| "This month vs last month" cards all show a big red drop | The data ends mid-month | Compare the latest *complete* month (see the measure template) |
| Numbers shown as K/M when whole numbers are wanted | Display units default to auto | `labelDisplayUnits = 1` on the value / labels |

## Environment

| Symptom | Cause | Fix |
|---|---|---|
| Random failures: "thread could not start", ECANCELED writes, installers failing with 1603 | System drive almost full | Check free space early (PowerShell `Get-PSDrive C`); ask the user before deleting anything; old installer caches in `%TEMP%\WinGet` are safe candidates |
| `powerbi-report-author` not found | Node.js missing or not on PATH | Reinstall Node.js LTS; refresh PATH in the shell |
| Git push asks for credentials / fails non-interactively | Credential helper (e.g. GitHub CLI) removed | Push from VS Code Source Control, or reinstall the credential helper |
