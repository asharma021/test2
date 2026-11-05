Here’s the Jira update covering everything since your 2026-11-03 comment. It’s plain text with no special characters, so it pastes cleanly into Jira.

2026-11-03 to 2026-11-05 update

CI/CD fixes (development)

MR !16: env_setup and dbt_compile now run automatically after merges into development, so pipelines no longer sit “Blocked”.
MR !18 / !19: dbt_evaluate was failing intermittently. The evaluator itself always passed, but the Windows PowerShell runner treated stderr output as a failure, so the Code Quality report step never ran. That’s fixed, and dbt_evaluate is green.
uv.lock conflict: merged production into development, keeping development’s uv.lock (no file changes). The development to production promotion now merges cleanly.
The remote-development fix MR was closed, since that work moved to development.

Promotion to test (MR !17)

Brought test up to the current design: compile deadlock fix, CI fixes, DEV/IVV extract/load work, Umair’s IVV config, Srini’s DI-4645 changes, and the prod target config.
The first test pipeline failed with “no password supplied”: DBT_PG_USER / DBT_PG_PASSWORD were scoped to “ivv”, while dbt jobs on the test branch use the “test” environment. Re-scoped them from ivv to test, and env_setup and dbt_compile then passed (pipelines #69702, #69704).
The run-stage dbt jobs on test depend on prod_mirror, so they wait until the mirror runs.

Production config changes (per Josh)

The prod target in profiles.yml is database oablmsdb, schema business_layer_prod.
postgres_extract_load on production: removed the manual step and the single-table default, so it loads all tables in tables.yml and runs automatically, including the scheduled pipeline.
Prod deploy permission granted to me (the prod environment is protected).

First production load (test to production promotion, pipeline #69708, job 199869)

The full first load ran for 6 hours and hit the job timeout.
Loaded into prod with watermarks set:
ITDS: itds_line 52.8M rows, itds_event_details 26.7M, itds_line_hts_list 17.8M+
myTTB: CBMA claims, chemist cert, export cert, ref data
ARR tables
All Tableau tables failed with a connection timeout from HWPOAAPP05 (GitLab runner Windows server) to hwptableau02:8060 (Tableau repository database). This is a network/firewall issue, not code or credentials, and it caused the job to run past 6 hours.

Scope change (Josh, fda2d1c)

ITDS and Tableau groups were commented out of tables.yml. ITDS data already loaded in prod is kept.
Promoted development to test (MR !21, pipeline #69859 passed), then test to production.
Production pipeline #69861: postgres_extract_load passed in about 5 minutes (incremental myTTB/ARR, plus the IVV groups).
Oracle, SQL Server and Informix extract/loads still fail (allowed to fail). To investigate separately.

Schedules (created by Josh)

Production daily incremental build: daily at 4:00 AM ET (extract_load_only).
Development weekly prod mirror: Saturdays at 3:00 PM ET.
Test weekly prod mirror: Saturdays at 1:00 PM ET.
Production monthly EL audit: second Sunday at 12:00 PM ET (verify-only).

Mirroring validation (prod to DEV)

prod_mirror failed with “Missing required env vars: MIRROR_PG_*”, because the job wasn’t pointed at the extract/load env file. Fixed by setting EL_ENV_FILE on prod_mirror, the same as postgres_extract_load (commit 68f4c88 on development).
The first mirror run connected and mirrored 56 tables. Most errors were tables that don’t exist in prod yet (UCCX, IRIS, DABL from the Oracle/SQL Server/Informix sources).
myTTB/ARR incremental tables were refused with a cursor mismatch: DEV had loaded them directly, tracking last_modified_date, while the mirror tracks _loaded_at. Fixed with a one-time full refresh of those 53 tables in DEV (job 
#200382).
Result: 46 of 53 tables mirrored from prod into DEV, 5,485,758 rows in 32 minutes. The largest were myttb_event_log 609,817 (plus 581,028 in the IVV copy), cbma_entry_line_item 1,799,680, cbma_claims_request_line_item 1,470,587, lexis_liens_details 138,740 and company_compliance_detail 136,011. Weekly runs will be incremental from now on.

Open items

7 tables fail mirroring with column "id" does not exist: myttb_message_details, myttb_message_thread, myttb_message_tracker, myttb_notification, myttb_bank_account, export_certificates, export_products. They inherit unique_key [id] from the shared incremental setting, but they have no id column. Waiting on Josh for the correct key columns, to update in tables.yml.
The mirror schedules may have prod_mirror and extract_load_only set as variables instead of inputs (the DEV schedule run left prod_mirror manual). To check.
TEST mirror: promote the prod_mirror fix to test, then run the TEST mirror.
Network request: HWPOAAPP05 to hwptableau02:8060 (plus Tableau repository access), then re-enable the Tableau group.
Decide when to re-enable ITDS in the daily run.
Investigate the Oracle, SQL Server and Informix extract/load failures in prod.

Next

Fix the 7 key columns, verify the schedule inputs, run the TEST mirror, and confirm the first scheduled 4:00 AM prod run.
