Promote development to test: E&L reliability (DI-4659, DI-4662, DI-4663, DI-4653)

## What & why

Promotes this week's E&L reliability work from development to test:

1. **E&L health check (DI-4659, DI-4663).** New `el_health_check` job fails scheduled/web pipelines when an E&L job or table fails unexpectedly, so failures are no longer hidden by `allow_failure` and GitLab's failed-pipeline email is sent. It writes `el_health_summary.md`, and the `pages` job publishes the summary with the site. Known failures are listed in `EL_EXPECTED_FAILURES` (currently `informix_extract_load`).
2. **Env pre-flight check (DI-4662).** `scripts/el_env_check.py` runs in `.el_job` right before the loader and fails fast with a clear message if the EL env file or a required setting is missing (e.g., `ORACLE_USER`). Values are never printed.
3. **Oracle config (DI-4653).** `DABL.VW_FACT_SOT_FST_RETURNS` commented out of `src/extract_load/oracle/tables.yml` (BL view; fails with ORA-01841). Already on production.

## Type of change

- [ ] dbt models / macros / tests / seeds
- [x] Extract/load (`src/extract_load/`, `tables.yml`)
- [x] CI/CD, tooling, or developer setup
- [ ] Reference Data Update
- [ ] Documentation only

## Data impact

- Models / sources affected: none. The new scripts only read logs and settings. `raw_bl_legacy.vw_fact_sot_fst_returns` is no longer extracted (it never loaded successfully).
- Needs a full refresh after merge? **No**
- Grain, key, or column renames that affect downstream consumers? **No**
- Validation on development:
  - Pipeline #71147 passed (setup and quality stages green).
  - Manual pipeline #71148: `el_health_check` ran on the runner and passed (all E&L jobs NOT RUN on dev, as expected).
  - CI syntax validated in the pipeline editor.

## Checklist

- [x] `uv run pre-commit run` passes (file hygiene, ruff, ty, SQLFluff, dbt parse, lock guards)
- [x] `uv run pyright` passes (`pyright_check` green in #71147)
- [x] `uv run bl evaluate` reviewed: N/A, no dbt model / DAG changes
- [x] New or changed models, macros, and columns are documented in YAML, with tests: N/A
- [x] Column names follow the abbreviation standards: N/A
- [x] README / reference docs / onboarding decks assessed and updated where needed: inline comments in `.gitlab-ci.yml` and script docstrings; runbook tracked in DI-4668
- [x] A significant architectural or tooling decision has an ADR in `docs/adr/`: N/A

Refs: DI-4659, DI-4662, DI-4663, DI-4653
