Promote development to test

Brings test up to the current development design. test was still on the old pipeline and folder layout. The latest MR pipeline (#69671) is fully green.

Pipeline and CI fixes
dbt_compile deadlock fixed (!9). The no_deprecated_abbreviations test queried seed_abbreviation_standards during dbt compile. Compile runs before dbt_seed, so it failed wherever that seed table didn’t exist yet. The query is now skipped during compile, and the full check still runs in dbt test / dbt build. This also prevents the same failure on the first IVV and prod runs.
No more stuck pipelines (!13). The only runner (#766) is protected, so pipelines from unprotected branches were never picked up. Unprotected branches and their MRs no longer create pipelines. Pipelines run once changes are merged into a protected branch.
No more “Blocked” after merges (!16). env_setup and dbt_compile now run automatically after merges into development.
dbt_evaluate fixed (!18, !19). The evaluator always passed, but the Windows PowerShell runner treated stderr output as a failure, so the Code Quality report step never ran. That step now runs and the job passes.
PostgreSQL extract/load
DEV/IVV extract/load jobs and config (!5). These include:
incremental load support;
verify-only runs;
IVV source configuration;
EL_CONFIG support in the shared extract/load job;
the read-only mirror readiness job.
Validated in DEV (incremental and rerun, 2,982 = 2,982, no duplicate IDs) and IVV (12 = 12, rerun safe).
Prod readiness (!6).
The prod target in dbt/profiles.yml is now database oablmsdb, schema business_layer_prod, host oa-prod-flexible-server.
postgres_extract_load is manual on production, with a single-table default for the first run.
IVV tables config (Umair). Unique indexes and incremental settings for the IVV tables in tables.yml, plus a new reportable_person_ownership_det table.
Permit module (Srini, DI-4645)
Updated ref data seed file for the permit module.
The quest data load changed from incremental to full.
Branch housekeeping
Merged production into development, keeping development’s uv.lock. This removes the only conflict, so the later promotion to production merges cleanly. No file content changes.
Validation
MR pipeline #69671 passed: env_setup, dbt_compile, uv_lock_check, dbt_evaluate, and all quality checks (freshness_columns, pyright_check, quality_report, ruff_lint, sqlfluff_lint, ty_check).
Merges cleanly into test.
What happens on merge
The test pipeline runs env_setup, dbt_compile, dbt_seed, dbt_run and dbt_test into IVV (business_layer_test) for the first time with the new design.
Extract/load jobs do not run on test.
Next
Promote test to production.
Run the first prod PostgreSQL extract/load: verify-only, then load, then verify.
Set up the production morning schedule.
Validate mirroring from prod into DEV.
