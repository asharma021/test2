Fail pipeline on Oracle/SQL Server/Postgres E&L failures; remove el_health_check (DI-4659)


## What & why

Per team review (11/18): remove the separate E&L health check job and instead make the production extract/load jobs fail the pipeline directly.

- `oracle_extract_load`, `sqlserver_extract_load`, `postgres_extract_load`: `allow_failure: false`. A failed load (any failed table) now fails the pipeline and stops the downstream dbt jobs.
- `informix_extract_load` and `prod_mirror` unchanged (still allowed to fail; Informix setup pending).
- Removed `el_health_check` (job and `scripts/el_health_check.py`), its `pages` lines, and the `.el_job` status after_script and artifact.
- `pages_audit` rules now use `when: always`, so the monthly verify-only audit page still publishes when it finds mismatches.
- Env pre-flight check (`el_env_check.py`, DI-4662) unchanged.

**Draft until:** the 3 `myttb_workflows792026_*` Postgres tables are fixed (source schema missing in prod), otherwise the first prod run fails on Postgres.

## Type of change

- [ ] dbt models / macros / tests / seeds
- [ ] Extract/load (`src/extract_load/`, `tables.yml`)
- [x] CI/CD, tooling, or developer setup
- [ ] Reference Data Update
- [ ] Documentation only

## Data impact

- Models / sources affected: none
- Needs a full refresh after merge? **No**
- Grain, key, or column renames? **No**
- Validation: YAML parsed before/after; only `.el_job`, `pages`, `pages_audit` and the three E&L jobs changed, and `el_health_check` was removed with no remaining references.

Refs: DI-4659
