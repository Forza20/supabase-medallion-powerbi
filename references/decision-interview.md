# Decision interview ("grill") – protocol and question bank

The pipeline in SKILL.md is mechanical; the hard part is the handful of modelling decisions that change
headline numbers. Those decisions belong to the user, because only they know what the numbers will be
used for. This file tells you how to put each decision to them, and which decisions to look for.

## Contents
1. Protocol
2. Decision log
3. Question bank
   - Round 1 – before building (purpose and shape)
   - Round 2 – after silver (source data quality)
   - Round 3 – after gold, before the report (meaning of the numbers)
   - Round 4 – before hand-over (exposure and operations)

---

## 1. Protocol

Follow this for every question. It is what makes the interview fast for the user and the answers
trustworthy.

1. **Investigate before you ask.** If the data or the code can answer part of the question, run the
   query first (detection SQL is given per question). Never ask the user something you could look up.
2. **Skip what does not apply.** If detection finds nothing (no versioned attributes, no missing dates),
   record "not applicable – checked" in the decision log and move on without asking.
3. **One question per message.** Wait for the answer before asking the next one. Later questions often
   depend on earlier answers (e.g. purpose decides how much operational hardening is worth it).
4. **Show the evidence.** State what you found in numbers: how many rows, how much revenue, what share
   of the total, and which headline figure it changes. Users decide well when they see the size of the
   effect.
5. **Offer 2–4 lettered options**, each with its consequence in one or two lines.
6. **Give your recommendation and why**, tied to the purpose agreed in question R1-1. Make it easy to
   accept ("your recommendation" / the letter is a complete answer).
7. **Confirm and log.** Restate the decision in one line, append it to the decision log, then ask the
   next question. Do not implement mid-interview unless the user asks; implement each round's decisions
   together, then rerun the quality checks.
8. If the environment provides a dedicated interview skill (for example one called `grill-me`), you may
   use it to run the conversation, but keep the evidence, options, recommendation and log described here.

Message shape (adapt the wording, keep the parts):

```
**Question N: <decision in plain words>?**

What the data shows:
- <finding with numbers>
- <which KPI/visual it affects and by how much>

Options:
- **A. ...** – consequence
- **B. ...** – consequence
- **C. ...** – consequence

**My recommendation: B.** <why, linked to the project's purpose>

Which do you choose?
```

## 2. Decision log

Keep the decisions in the project, e.g. `docs/model-decisions.md`, so the next person (or the next
agent session) knows why the numbers are what they are. Template:

```markdown
# Model decisions

| # | Decision | Evidence | Choice | Consequence | Implemented |
|---|---|---|---|---|---|
| R1-1 | Purpose of the project | – | Demo/reference | Correct, explainable numbers over automation | – |
| R2-1 | Cost for margin | 77 products changed cost; 19,054 of 60,398 lines | Point-in-time cost on fact | Margin 39.8% → 43.0% | yes, migration 0007 |
```

Mention the log in the hand-over and in the user guide.

## 3. Question bank

Each entry: why it matters → how to detect → typical options → default recommendation. Adapt names to
the user's tables. Ask only the questions whose detection finds something (except Round 1, which is
always asked).

### Round 1 – before building (always ask)

**R1-1 Purpose and audience.** Decides every trade-off that follows.
- Options: A demo/reference project (static data, explainable numbers) · B template for real client
  projects (reusable, must fail loudly) · C production for one organisation (scheduled refresh, access
  control, monitoring).
- Recommend from context: static historical data and no refresh requirement → A.

**R1-2 Grain of each fact table.** Wrong grain breaks every measure.
- Detect: for candidate keys, `select key_cols, count(*) from bronze.x group by 1.. having count(*) > 1`.
- Options: per transaction line · per document/order · per day/period snapshot.
- Recommend the lowest grain the sources support (usually the line).

**R1-3 Business keys and how sources join.** Different systems often format keys differently.
- Detect: sample keys from each source side by side; test join rates
  (`count(*) filter (where b.key is null)` after a left join).
- Options: normalise in silver (strip prefixes/separators) · map table · accept unmatched as unknown.
- Recommend normalising in silver, with a test that reports the unmatched count.

**R1-4 History in dimensions.** Versioned rows (start/end dates) need a policy.
- Detect: `select key, count(*) from silver.dim_source group by 1 having count(*) > 1`.
- Options: current version only (SCD1) · full history with version keys (SCD2) · current dimension plus
  point-in-time values on the fact (see R2-1).
- Recommend SCD1 dimensions plus point-in-time facts unless users must filter by historical attribute
  values.

**R1-5 Refresh cadence.** One-off load, manual reload or scheduled.
- Options: manual (`call` procedures + Refresh) · `pg_cron` nightly · external orchestrator.
- Recommend manual for A, `pg_cron` for B/C.

### Round 2 – after silver (ask when detection finds something)

**R2-1 Historical prices/costs on versioned dimensions.** Using today's cost for old sales misstates
margin, often by several points.
- Detect:
  ```sql
  select count(*) as keys_with_changing_value
  from (select prd_key from silver.crm_products group by 1 having count(distinct prd_cost) > 1) x;
  -- plus: how many fact lines reference those keys, and margin with current vs point-in-time cost
  ```
- Options: A current cost, documented · B SCD2 dimension · C store `unit_cost` on the fact, looked up at
  the transaction date (gap rule: latest version started before the date, else the first version).
- Recommend C when margin is a headline KPI. Template: `sql-templates.md` → *Point-in-time value*.

**R2-2 Missing or invalid dates.** Rows without a date drop out of every time-based visual and of
YTD/YoY, while still counting in totals.
- Detect: `count(*) where order_date is null`; then look for a deterministic pattern, e.g.
  `select min(ship - order), max(ship - order)` and sibling lines in the same document.
- Options: A leave empty, documented · B derive from siblings, then from a proven lag, and flag with
  `<col>_derived` · C drop rows (breaks reconciliation with the source).
- Recommend B only if the pattern holds for (nearly) all other rows; otherwise A.
  Template: `sql-templates.md` → *Derived date with flag*.

**R2-3 Conflicts between sources for the same attribute.** E.g. gender in CRM vs ERP.
- Detect: join the sources and count rows where both are known and differ.
- Options: A one master system per attribute, fall back to the other, count conflicts in an info check ·
  B the other system as master · C set conflicts to unknown.
- Recommend A with the system where the record is created/maintained as master.

**R2-4 Unknown members.** NULL vs `'n/a'` vs an "Unknown" dimension row.
- Detect: share of NULL/blank per attribute and per foreign key.
- Options: `'n/a'` text in attributes · unknown member row (key -1) for missing foreign keys · leave NULL.
- Recommend `'n/a'` for attributes and an unknown member when fact foreign keys can be missing, so
  totals always reconcile.

**R2-5 Duplicates.** Which row wins when a key repeats.
- Detect: duplicate count per business key in bronze.
- Options: latest by created/modified date · first · reject and report.
- Recommend latest, with the dropped count reported.

### Round 3 – after gold, before the report

**R3-1 Non-additive counts in breakdowns.** Distinct counts (orders, customers) per category do not sum
to the total, because one order can contain several categories.
- Detect: compare the KPI total with the sum over the breakdown; count documents spanning several
  members.
- Options: A keep the count and name it precisely ("Orders containing the category") · B switch to an
  additive measure (lines, revenue) · C show share of orders.
- Recommend A when the question "how many orders include X" is meaningful; fix the title, not the number.

**R3-2 Incomplete periods at the edges of the data.** A first/last month with only a few days (or a
partial extract) looks like a collapse or a spike in trend visuals and distorts YoY.
- Detect:
  ```sql
  select date_trunc('month', order_date) m, min(order_date), max(order_date), sum(sales_amount), count(*)
  from gold.fact_sales group by 1 order by 1;  -- inspect first and last two months
  ```
  Also compare the last month's revenue per day and average order value with the month before – a big
  drop with a nearly full month suggests an incomplete extract, not a short month.
- Options: A show everything, add a note · B `is_complete_month` in the date table; time visuals filter
  to complete months, totals keep everything · C remove the months (breaks reconciliation).
- Recommend B, and document any anomaly. Template: `tmdl-templates.md` → *Complete-month flag*.

**R3-3 "Current period" cards.** Month-over-month cards need a defined current month.
- Options: latest complete month · latest month with data · a user-selected month (slicer).
- Recommend latest complete month (measure template in `tmdl-templates.md`).

**R3-4 Units, currency and rounding.** Whole numbers vs K/M, currency symbol, decimals on percentages.
- Recommend whole numbers with thousands separators for executive pages; one decimal on percentages.

**R3-5 Replacing KPIs the data cannot support.** A layout or brief may ask for metrics the sources do not
have (returns, targets, profit without cost).
- Options: substitute a supported KPI (e.g. margin %) · leave a placeholder · source the data.
- Recommend substituting and saying so explicitly in the hand-over.

### Round 4 – before hand-over

**R4-1 Repository visibility and identifiers.** Project ref, pooler host and user names are not secrets
on their own, but together with a public repo they leave only the password between a stranger and the
data.
- Detect: is the remote public (an unauthenticated `git ls-remote` succeeds)? `git grep` for the project
  ref, pooler host and `<role>.<ref>` user names.
- Options: A make the repo private · B stay public, replace identifiers with placeholders/parameters,
  strong password, consider Supabase network restrictions · C leave as is.
- Recommend A unless the repo is a public portfolio; then B. Old values remain in Git history.

**R4-2 Password and access.** Who holds the `powerbi_reader` password, and who else needs access
(Power BI Service, colleagues)?
- Recommend a long generated password stored in a password manager; separate roles per consumer.

**R4-3 Documentation.** Who reads the guide, in which language, and where it lives.
- Recommend a Markdown guide in the repo plus the decision log.
