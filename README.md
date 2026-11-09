1. Oracle extract/load to prod

Description: Validate the first prod load of the Oracle tables (DABL, IRIS, blpermit/gprocess) in the daily 4 AM build. Check run time against the job time limit and review the iris_trade_names_mv row-count difference.

Acceptance criteria:

oracle_extract_load succeeds in the 4 AM prod build with 0 table errors
All 164 tables land in the prod raw schemas
Run time stays within the 6-hour job limit
iris_trade_names_mv difference reviewed and resolved
2. SQL Server extract/load to prod

Description: Validate the first prod load of the SQL Server tables (RDM, quest, autoaudit) and confirm the daily incremental runs.

Acceptance criteria:

sqlserver_extract_load succeeds in the 4 AM prod build with 0 table errors
All 10 tables land in the prod raw schemas
Next day’s run loads incrementally
3. Informix (UCCX) extract/load to prod

Description: Set up the Informix extract/load in prod. Get the UCCX connection details from Ben, get the Informix ODBC driver installed on app05 by PS, update the env file and validate the load.

Acceptance criteria:

Real UCCX connection details in the env file
Informix ODBC driver installed on app05 (RFC if needed)
informix_extract_load succeeds in prod with all 23 tables loaded
4. Tableau extract/load connectivity

Description: Resolve the connection timeouts from app05 to the Tableau repository server with Ben, then re-enable and validate the Tableau load.

Acceptance criteria:

Network access from app05 to the Tableau repository server confirmed
Tableau group re-enabled in tables.yml
Tableau tables load in prod without timeouts
5. Re-enable ITDS extract/load

Description: Agree the ITDS load approach with Josh, re-enable it and confirm it completes within the job time limit.

Acceptance criteria:

Load approach agreed (full vs incremental, timing)
ITDS group re-enabled in tables.yml
ITDS load finishes in prod within the time limit
6. Schedule inputs and weekly mirror validation

Description: Set the schedule inputs correctly so the daily build runs only E/L and the Saturday schedules run the prod mirror. Confirm the new Oracle and SQL Server tables reach DEV and TEST.

Acceptance criteria:

extract_load_only and prod_mirror set as inputs on the schedules
Saturday TEST and DEV mirrors complete successfully
Oracle and SQL Server tables present in DEV and TEST after the mirror
7. E/L failure notifications

Description: Send an alert when a scheduled extract/load or mirror job fails.

Acceptance criteria:

Failed scheduled jobs send a notification (email or Teams)
Notification includes job name, branch and log link
Tested with a forced failure
8. Retry with backoff and checkpointing

Description: Retry failed tables automatically with backoff, and let reruns resume from where the last run stopped. With Umair.

Acceptance criteria:

Failed tables retry with exponential backoff
Reruns skip tables that already succeeded
Tested with a simulated connection failure
9. Post-load row-count validation and monthly audit

Description: Compare source and target row counts after each load, and confirm the monthly verify-only audit runs. With Umair.

Acceptance criteria:

Each load flags count mismatches
Monthly audit runs on schedule and publishes its report
Known differences documented or resolved
10. Env setting integrity check

Description: Fail the job right away when required connection settings are missing or still placeholders.

Acceptance criteria:

Required settings checked before connecting
Missing or placeholder values fail the job with a clear message
Setting values never printed in the log
11. Daily E/L health summary

Description: Publish a summary after each daily build with duration, tables, rows and errors per source.

Acceptance criteria:

Summary generated automatically after the 4 AM build
Covers every source
Shared through an agreed channel
12. Permit Mod transformations in the prod pipeline

Description: Add the Permit Mod dbt build and tests to the prod pipeline after extract/load.

Acceptance criteria:

dbt build and tests run in prod after a successful E/L
Permit Mod models build without errors
A failed E/L stops the dbt run
13. E/L to Oracle destination

Description: Load the transformed Permit Mod data into Oracle after the dbt run. Depends on Yves / Asad.

Acceptance criteria:

Target Oracle schema and access confirmed
Transformed tables load into Oracle in the pipeline
Row counts match between Postgres and Oracle
14. Downstream refresh automation

Description: Trigger the refresh of downstream data sources (Tableau, SAS) after the Postgres and Oracle transformations finish.

Acceptance criteria:

Downstream sources and dependencies documented
Refresh triggered automatically after transformations
Refresh failures reported
15. Dev → test → prod promotion automation

Description: Standardize promotion between branches with required checks.

Acceptance criteria:

Promotion steps and checks documented
Required checks enforced before merging to test and production
Process reviewed with the team
16. E/L runbook and PS handoff

Description: Write a runbook for the extract/load pipeline and hand it over to PS.

Acceptance criteria:

Runbook covers config files, schedules, adding a table, the mirror and common errors
Walkthrough held with PS
PS confirms ownership of the wider E/L work
