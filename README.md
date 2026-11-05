2026-11-05 update (continued)

Schedules (created by Josh)

Production daily incremental build: daily at 4:00 AM ET (extract_load_only).
Development weekly prod mirror: Saturdays at 3:00 PM ET.
Test weekly prod mirror: Saturdays at 1:00 PM ET.
Production monthly EL audit: second Sunday at 12:00 PM ET (verify-only).

Mirroring fixes

prod_mirror failed with “Missing required env vars: MIRROR_PG_*” because the job had no env file. It now uses the same EL_ENV_FILE as postgres_extract_load (commit 68f4c88).
DEV tables that had been loaded directly earlier used a different change-tracking column (last_modified_date vs the mirror’s _loaded_at), so the mirror refused them. Fixed with a one-time full refresh of those tables in DEV (job 
#200382: 46 of 53 tables, 5,485,758 rows).
7 tables failed with column "id" does not exist. They inherited unique_key [id] from the shared incremental settings but have no id column. Using the key columns Josh provided (message_details_id, message_thread_id, message_tracker_id, notification_id, bank_account_id, certificate_id, export_product_id), updated tables.yml (commit f50b6d6). The DEV re-mirror of those 7 tables succeeded (job 
#200488, 0 errors).

CI/CD cleanup (per Josh)

Removed the temporary DEV/IVV validation jobs: postgres_extract_load_dev_incremental, postgres_extract_load_ivv_incremental, postgres_extract_load_dev_verify, postgres_extract_load_test, prod_mirror_dev_verify, and the .postgres_incremental_guard template.
Removed tables_ivv_verify.yml. All other jobs are unchanged (commit a6f9c7e). Development pipeline #69919 passed.

Promotion to test (MR !23)

Includes the mirror env file fix, the 7 key columns, and the CI cleanup. MR pipeline #69930 passed, and the test pipeline (#69944) passed.

Mirroring validation results

DEV (prod → DEV): all PostgreSQL tables that exist in prod mirror successfully.
TEST (prod → IVV): all PostgreSQL tables that exist in prod mirror successfully, with no cursor errors.
Examples: myttb_event_log 581,028 / 609,817 rows, cbma_entry_line_item 1,799,680, cbma_claims_request_line_item 1,470,587, myttb_workflows792026_event_log 432,523, lexis_liens_details 138,740, company_compliance_detail 136,011, the 7 key-column tables, controlled group, auth, chemist cert, export cert.
The first full copy took about 57 minutes; weekly runs will be incremental.
The only mirror failures are tables not yet in prod, from the Oracle, SQL Server and Informix sources (autoaudit, DABL, IRIS, UCCX, blpermit/gprocess). Expected until those sources load into prod.

Requests sent

Ben (PS): network access from HWPOAAPP05 (GitLab runner Windows server) to hwptableau02:8060 (Tableau repository database), plus Tableau repository access, so the Tableau group can be re-enabled.

Next

Check the first scheduled 4:00 AM ET production run.
Promote test to production (key columns, mirror fix, cleanup), merging with [skip ci].
Confirm the mirror schedules use inputs (prod_mirror, extract_load_only), not variables.
Oracle, SQL Server and Informix extract/load into prod (new scope from Josh): diagnose the failures from the production job logs and fix them.
Re-enable Tableau once network access is open. Decide with Josh when to re-enable ITDS in the daily run.
