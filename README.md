E&L reliability: failure alerts, env pre-flight check, Oracle view sync



## What & why

Three changes so E&L failures are caught early and reported instead of hidden:

1. **E&L health check (DI-4659, DI-4663).** E&L jobs use `allow_failure: true`, so a failed job only shows as a yellow warning and nobody is notified (e.g., the Oracle failure on 11/10 was found only by manual checking).
   - `.el_job` now has an `after_script` that records each job's final status in `logs/<job>.status` (kept as an artifact). `after_script` cannot fail the job.
   - New `el_health_check` job (`.post` stage, schedule and web pipelines only) reads each E&L job's log and status, writes `el_health_summary.md` (job, result, tables, rows, failed tables), and **fails the pipeline** when an E&L job or table failed unexpectedly, which triggers GitLab's standard failed-pipeline email.
   - Known failures are listed in `EL_EXPECTED_FAILURES` (currently `informix_extract_load`, until its source is set up). Jobs that did not run are reported as `NOT RUN` and do not fail the check.

2. **Env pre-flight check (DI-4662).** From 11/5 to 11/8 every prod Oracle run failed with a bare `'ORACLE_USER'` error because a setting was missing from the EL env file.
   - New `scripts/el_env_check.py` runs in `.el_job` right before the loader. It loads the EL env file the same way the loaders do and fails fast with a clear message when the file is not usable, a required source setting (e.g., `ORACLE_USER`) is missing or empty, or the load target login/endpoint cannot be resolved.
   - Values are never printed. Errors are also written to the job log, so `el_health_check` reports them.
   - Only fails on conditions that would crash the loader anyway. A missing base setting with only profile variants present (e.g., `MSSQL_USER_QUEST`) is a warning. `EL_ENV_CHECK=off` bypasses the check.

3. **Sync from production (DI-4653).** Cherry-pick of the Oracle config change already merged to `production`: comments out `DABL.VW_FACT_SOT_FST_RETURNS` in `src/extract_load/oracle/tables.yml` (BL view, fails with ORA-01841). `tables.yml` here is identical to production, so promotion will not conflict.

## Type of change

- [ ] dbt models / macros / tests / seeds
- [x] Extract/load (`src/extract_load/`, `tables.yml`)
- [x] CI/CD, tooling, or developer setup
- [ ] Reference Data Update
- [ ] Documentation only

## Data impact

- Models / sources affected: none. Both new scripts only read logs/settings. `raw_bl_legacy.vw_fact_sot_fst_returns` is no longer extracted (it never loaded successfully).
- Needs a full refresh after merge? **No**
- Grain, key, or column renames that affect downstream consumers? **No**
- Validation (local):
  - `el_health_check.py`: 4 scenarios (Oracle table failure → fails; clean run → passes; nothing ran → passes; failed table inside a successful job → fails).
  - `el_env_check.py`: 6 scenarios using the real loader code (all set → passes; `ORACLE_USER` missing → fails; env file missing → fails; wrong source settings → fails; profile variants only → warns and passes; `EL_ENV_CHECK=off` → skipped). Confirmed secret values never appear in output and errors reach the job log.
  - `.gitlab-ci.yml` parsed before/after each change: only `.el_job` (after_script, status artifact, env-check line) and the new `el_health_check` job changed; all other jobs unchanged.

## Checklist

- [x] `uv run pre-commit run` passes (file hygiene, ruff, ty, SQLFluff, dbt parse, lock guards)
- [ ] `uv run pyright` passes (new Python scripts; covered by the MR pipeline `pyright_check`)
- [x] `uv run bl evaluate` reviewed: N/A, no dbt model / DAG changes
- [x] New or changed models, macros, and columns are documented in YAML, with tests: N/A
- [x] Column names follow the abbreviation standards: N/A
- [x] README / reference docs / onboarding decks assessed and updated where needed: inline comments in `.gitlab-ci.yml` and script docstrings; runbook update tracked in DI-4668
- [x] A significant architectural or tooling decision has an ADR in `docs/adr/`: N/A

Refs: DI-4659, DI-4662, DI-4663, DI-4653
