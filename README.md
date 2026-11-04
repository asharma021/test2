## Promote test to production

Brings `production` up to the current design (production is still on the old pipeline). Merges cleanly; the uv.lock conflict was resolved earlier by merging production into development.

### Includes
- Everything in !17: compile deadlock fix, CI fixes (no stuck or blocked pipelines, dbt_evaluate), DEV/IVV extract/load work, Umair's IVV config, Srini's DI-4645 changes.
- Prod target: database `oablmsdb`, schema `business_layer_prod`, host `oa-prod-flexible-server`.
- PostgreSQL extract/load on production loads all tables in `tables.yml` and runs automatically.

### Validation
- Test pipeline #69702 / #69704: `env_setup` and `dbt_compile` passed against IVV with the test-scoped credentials.
- Credentials: `DBT_PG_USER` / `DBT_PG_PASSWORD` scoped to `prod` exist (confirmed by PS).

### What happens on merge
- The production pipeline starts automatically and runs all extract/loads into prod: PostgreSQL (all tables), Oracle, SQL Server, Informix, followed by the dbt jobs into prod.

### Next
1. Verify the PostgreSQL load in prod (row counts vs source).
2. Set up the production morning schedule.
3. Validate mirroring from prod into test/development (`prod_mirror = true`), which also runs the dbt build in the lower environments.
