Story 1: Oracle extract/load to prod

As a data analyst, I want the Oracle source tables loaded into prod daily, so that DABL, IRIS and permit data are available in the business layer.

Acceptance criteria:

oracle_extract_load succeeds in the 4 AM prod build with 0 table errors
All 164 tables land in their prod raw schemas
Run time stays within the 6-hour job limit
iris_trade_names_mv row-count difference reviewed and resolved
Story 2: SQL Server extract/load to prod

As a data analyst, I want the SQL Server source tables loaded into prod daily, so that RDM, quest and autoaudit data are available in the business layer.

Acceptance criteria:

sqlserver_extract_load succeeds in the 4 AM prod build with 0 table errors
All 10 tables land in their prod raw schemas
The daily incremental runs pick up new rows
Story 3: Informix (UCCX) extract/load to prod

As a data analyst, I want the UCCX contact-center tables loaded into prod daily, so that call center data is available in the business layer.

Acceptance criteria:

Real UCCX connection details received from Ben and added to the env file
Informix ODBC driver installed on app05 by PS (RFC if needed)
informix_extract_load succeeds in prod with all 23 tables loaded
Story 4: Tableau extract/load connectivity

As a data analyst, I want the Tableau repository data loaded into prod, so that Tableau usage data is available in the business layer.

Acceptance criteria:

Network access from app05 to the Tableau repository server is in place (Ben)
Tableau group re-enabled in tables.yml
Tableau tables load in prod without connection timeouts
Story 5: Re-enable ITDS extract/load

As a data analyst, I want ITDS data loaded into prod, so that ITDS data is available in the business layer.

Acceptance criteria:

Load approach agreed with Josh (full vs incremental, timing)
ITDS group re-enabled in tables.yml
ITDS load finishes within the job time limit in prod
Story 6: Schedule inputs and weekly mirror validation

As a developer, I want the scheduled pipelines to run exactly what they’re meant to, so that prod loads and DEV/TEST mirrors happen reliably without extra jobs.

Acceptance criteria:

extract_load_only and prod_mirror set as inputs on the daily and weekly schedules
Saturday TEST and DEV mirror runs complete successfully
New Oracle and SQL Server tables appear in DEV and TEST after the mirror
Story 7: E/L failure notifications

As a team member, I want to be alerted when a scheduled E/L or mirror job fails, so that issues are fixed before users are affected.

Acceptance criteria:

A failed scheduled E/L or mirror job sends a notification (email or Teams)
The notification includes the job name, branch and a link to the log
Tested with a forced failure
Story 8: Retry with backoff and checkpointing

As a developer, I want failed tables retried automatically and runs to resume where they stopped, so that short outages don’t require a full manual rerun.

Acceptance criteria:

Failed tables retry with exponential backoff
A rerun skips tables that already succeeded in the same run
Tested with a simulated connection failure (with Umair)
Story 9: Post-load row-count validation and monthly audit

As a data analyst, I want row counts checked after each load and audited monthly, so that missing or duplicated data is caught early.

Acceptance criteria:

Each load compares source and target counts and flags mismatches
The monthly verify-only audit runs on schedule and publishes its report
Known differences documented or resolved (with Umair)
Story 10: Env setting integrity check

As a developer, I want E/L jobs to stop right away when connection settings are missing or still placeholders, so that config problems are obvious instead of failing table by table.

Acceptance criteria:

The job checks the required settings for its source before connecting
Missing or placeholder values fail the job with a clear message naming the setting
Values are never printed in the log
Story 11: Daily E/L health summary

As a team lead, I want a daily summary of each E/L run, so that I can see load health at a glance.

Acceptance criteria:

The summary lists each source with duration, tables, rows loaded and errors
Published automatically after the 4 AM build
Shared with the team through an agreed channel
Story 12: Permit Mod transformations in the prod pipeline

As a data analyst, I want the Permit Mod dbt models to run in prod after E/L, so that transformed permit data is always current.

Acceptance criteria:

dbt build and tests run in prod after a successful E/L
Permit Mod models build without errors
A failed E/L stops the dbt run
Story 13: E/L to Oracle destination

As a data consumer, I want the transformed Permit Mod data loaded into Oracle, so that downstream Oracle-based systems can use it.

Acceptance criteria:

Target Oracle schema and access confirmed with Yves / Asad
Transformed tables load into Oracle after the dbt run
Row counts match between Postgres and Oracle
Story 14: Downstream refresh strategy

As a report user, I want Tableau and SAS data sources refreshed after the transformations finish, so that reports show the latest data.

Acceptance criteria:

Downstream data sources and their refresh dependencies documented
Refresh triggered automatically after the Postgres and Oracle transformations
Refresh failures are reported
Story 15: Dev → test → prod promotion automation

As a developer, I want a standard promotion process with automated checks, so that releases are consistent and don’t depend on manual steps.

Acceptance criteria:

Promotion steps and required checks documented
Required checks enforced before merging to test and production
Promotion process walked through with the team
Story 16: E/L runbook and PS handoff

As a PS team member, I want a runbook for the E/L pipeline, so that I can support and extend it on my own.

Acceptance criteria:

Runbook covers config files, schedules, adding a table, the mirror and common errors
Walkthrough session held with PS
PS confirms they can own the wider E/L work
