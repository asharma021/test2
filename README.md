## What & why

Comments out `DABL.VW_FACT_SOT_FST_RETURNS` in `src/extract_load/oracle/tables.yml`.

This is a BL view, not a source table, and should not be in the Oracle E&L. Its `Rcvd_Date` logic (`TO_DATE(SUBSTR(CONTROL_NO, ...), 'YYYYDDD')`) fails with **ORA-01841** when the full view is read, which fails `oracle_extract_load` on every scheduled prod run (e.g., pipeline #70387, job #202094). Agreed with Josh/Yves to comment it out until the view SQL is fixed (backlog).

Ref: DI-4653

## Type of change

- [ ] dbt models / macros / tests / seeds
- [x] Extract/load (`src/extract_load/`, `tables.yml`)
- [ ] CI/CD, tooling, or developer setup
- [ ] Reference Data Update
- [ ] Documentation only

## Data impact

- Models / sources affected: `raw_bl_legacy.vw_fact_sot_fst_returns` is no longer loaded. It has never loaded successfully in prod (it failed on both runs since Oracle credentials were configured on 11/9), so there is no change to existing data. Downstream EA transformations that use this view stay without it until the view SQL is fixed.
- Needs a full refresh after merge? **No**
- Grain, key, or column renames that affect downstream consumers? **No**
- Queries run to validate: none needed (config-only change). Parsed `tables.yml` before and after: 164 → 163 tables, only `VW_FACT_SOT_FST_RETURNS` removed.

## Checklist

- [ ] `uv run pre-commit run` passes (file hygiene, ruff, ty, SQLFluff, dbt parse, lock guards)
- [x] `uv run pyright` passes: N/A, no Python changes
- [x] `uv run bl evaluate` reviewed: N/A, no dbt model / DAG changes
- [x] New or changed models, macros, and columns are documented in YAML, with tests: N/A
- [x] Column names follow the abbreviation standards: N/A
- [x] README / reference docs / onboarding decks assessed and updated where needed: inline comment added in `tables.yml`; README unchanged
- [x] A significant architectural or tooling decision has an ADR in `docs/adr/`: N/A
