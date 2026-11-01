# Progress Log


## Day 1 - 28 Oct 2026:: Setup and first ingestion
- Created the Fabric workspace and Lakehouse `lh_nyc_mobility`, with Bronze folders for taxi, weather and zones
- Built `pl_ingest_zones` (Copy activity, HTTP source to Lakehouse Files, Binary format)
- Built `pl_ingest_taxi` and parameterized it with `year_month` (dynamic relative URL and file name)
- Loaded Jan and Feb 2024 taxi files into Bronze as a parameter test
- **Problem:** the GitHub option in Fabric Git integration was greyed out. The admin tenant setting that allows it is off, and I don't have tenant admin rights.
  **Decision:** keep GitHub as the public repo and export notebooks and pipeline JSON manually after each session.
- Next: weather pipeline, config table, master pipeline

## Day 2 - 29 Oct 2026:: Metadata-driven orchestration
- Created `ops.pipeline_config` (months, date ranges, `is_active` flag) with a notebook
- Built `pl_ingest_weather` (REST/HTTP source, parameters for start and end date)
- **Problem:** the weather pipeline failed with HTTP 400 when the URL was built with dynamic content.
  **Investigation:** a hard-coded URL worked, so the connection was fine. The dynamic version contained an encoded timezone value (`%2F`) that the expression altered.
  **Fix:** used `timezone=auto`, and the pipeline succeeded.
- Built `pl_master_ingest` with Lookup (config table) and ForEach calling the taxi and weather child pipelines
- Result: [ ] months of taxi and weather files landed in Bronze in a single run

## Day 3 - 30 Oct 2026: Silver layer
- Built the Silver transformations in PySpark: type casting, derived columns (duration, pickup hour, tip %), and a hash-based `trip_id` as the business key
- Data quality rules with a rejects table (`ops.dq_rejects`): null timestamps, drop-off before pickup, pickup outside the file's month, non-positive fare, bad distance, trips over 6 hours, missing zones
- Loaded Silver with Delta MERGE, so reruns don't create duplicates (rerun test: counts unchanged: [ ])
- Flattened the nested weather JSON (hourly arrays) into one row per hour
- Added `ops.pipeline_log` (step, month, rows in, rows good, rows rejected, status, message, timestamp)
- Result: [ ] rows loaded, [ ] rejected. Top reject reason: [ ]
- **Problem:** `unsupported operand types` error in the taxi cell.
  **Cause:** a DataFrame was divided or multiplied by a number instead of a column.
  **Fix:** rewrote the cell as separate steps so each error points to a single line.
- **Problem:** Silver looked empty after a run.
  **Fix:** checked the log and the reject counts by reason before changing any rule.

## Day 4 - 31 Oct 2026: Orchestration issue, Gold layer and report
- **Problem:** the pipeline Notebook activity failed with `Failed to get Workspace Identity access token ... PowerBIEntityNotFound`.
  The notebook ran fine when I ran it manually, and the error appeared before any code executed, so the cause was the connection, not the code.
  The connection used Workspace identity or Service principal authentication, which needs admin-level setup I don't have on this trial tenant.
  **Decision:** keep Data Factory for ingestion and orchestration, and run the transformation notebooks directly. The Silver notebook loops over every active month in `ops.pipeline_config`, and failed months are logged without stopping the rest.
- Built the Gold star schema: `fact_trips`, `dim_date`, `dim_zone`, `dim_weather`, `dim_payment_type`
- Added logging to the Gold notebook and referential (orphan key) checks on the fact table
- **Problem:** Delta schema mismatch writing to `ops.pipeline_log` (a column was named `_8` instead of `run_ts`).
  **Fix:** dropped and recreated the log table, and made logging failures non-fatal.
- **Problem:** the orphan check failed with a missing attribute error.
  **Cause:** two references to the same table in one DataFrame join.
  **Fix:** rewrote the checks as SQL LEFT JOINs.
- Built the semantic model `sm_nyc_mobility` (relationships, 5 DAX measures) and the report `rpt_nyc_mobility`
- **Problem:** "fields that don't have a supported set of relationship" error in visuals.
  **Cause:** dimensions only relate through the fact table, so every visual needs a measure from `fact_trips`.
  **Fix:** added a fact measure to each affected visual.
- Used a "trips per weather hour" measure to compare rainy and dry periods fairly, since there are more dry hours
- Key finding: [ ]

## Day 5 - 01 Nov 2026: Documentation
- Exported notebooks and pipeline JSON to GitHub
- Added `docs/dax-measures.md`, `docs/data-model.md` and screenshots
- Wrote the README (architecture, design decisions, data quality rules, limitations)
- Closed finished issues

---

## Known limitations and what I'd change in production
- Notebook activity needs workspace identity or service principal authentication, which wasn't available on my trial tenant. In production I'd call the notebooks from the pipeline.
- Gold tables are fully rebuilt on each run. In production I'd make them incremental.
- Git sync was unavailable, so exports were manual. In production I'd use Fabric Git integration and deployment pipelines.
- No alerting yet. I'd add failure notifications.
