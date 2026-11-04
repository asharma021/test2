## Promote development to test

Brings `test` up to the current development design (`test` was still on the old pipeline and folder layout). Latest MR pipeline #69671 is fully green.

### Pipeline and CI fixes

- **dbt_compile deadlock fixed (!9)**
  - The `no_deprecated_abbreviations` test queried `seed_abbreviation_standards` during `dbt compile`.
  - Compile runs before `dbt_seed`, so it failed wherever that seed table did not exist yet.
  - The query is now skipped during compile; the full check still runs in `dbt test` / `dbt build`.
  - Also prevents the same failure on the first IVV and prod runs.
- **No more stuck pipelines (!13)**
  - The only runner (#766) is protected, so pipelines from unprotected branches were never picked up.
  - Unprotected branches and their MRs no longer create pipelines; pipelines run after merge into a protected branch.
- **No more "Blocked" after merges (!16)**
  - `env_setup` and `dbt_compile` now run automatically after merges into development.
- **dbt_evaluate fixed (!18, !19)**
  - The evaluator always passed, but the Windows PowerShell runner treated stderr output as a failure, so the Code Quality report step never ran.
  - The report step now runs and the job passes.

### PostgreSQL extract/load

- **DEV/IVV extract/load jobs and config (!5)**
  - Incremental load support, verify-only runs, IVV source config.
  - `EL_CONFIG` support in the shared extract/load job.
  - Read-only mirror readiness job.
  - Validated in DEV (incremental + rerun, 2,982 = 2,982, no duplicate IDs) and IVV (12 = 12, rerun safe).
- **Prod readiness (!6)**
  - Prod target in `dbt/profiles.yml`: database `oablmsdb`, schema `business_layer_prod`, host `oa-prod-flexible-server`.
  - `postgres_extract_load` is manual on production, with a single-table default for the first run.
- **IVV tables config (Umair)**
  - Unique indexes and incremental settings for the IVV tables in `tables.yml`.
  - New table `reportable_person_ownership_det`.

### Permit module (Srini, DI-4645)

- Updated ref data seed file for the permit module.
- Quest data load changed from incremental to full.

### Branch housekeeping

- Merged `production` into `development`, keeping development's `uv.lock`.
- Removes the only conflict, so the later promotion to production merges cleanly. No file content changes.

### Validation

- MR pipeline #69671 passed: `env_setup`, `dbt_compile`, `uv_lock_check`, `dbt_evaluate`, `freshness_columns`, `pyright_check`, `quality_report`, `ruff_lint`, `sqlfluff_lint`, `ty_check`.
- Merges cleanly into `test`.

### What happens on merge

- The `test` pipeline runs `env_setup`, `dbt_compile`, `dbt_seed`, `dbt_run`, `dbt_test` into IVV (`business_layer_test`) for the first time with the new design.
- Extract/load jobs do not run on `test`.

### Next

1. Promote `test` to `production`.
2. First prod PostgreSQL extract/load: verify-only, then load, then verify.
3. Set up the production morning schedule.
4. Validate mirroring from prod into DEV.
