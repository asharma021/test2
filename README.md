## Summary

Adds EL_ENV_FILE to oracle_extract_load, sqlserver_extract_load and informix_extract_load (same env file path as postgres_extract_load). Without it, those jobs could not read their source connection settings and failed in prod on missing INFORMIX_HOST / MSSQL_USER. The temporary dev verify-only rule used to test this has already been reverted, so it is not part of this change.

## Type of change

- [ ] dbt models / macros / tests / seeds
- [x] Extract/load (`src/extract_load/`, `tables.yml`)
- [x] CI/CD, tooling, or developer setup
- [ ] Reference Data Update
- [ ] Documentation only

## Data impact

- Models / sources affected: No model changes. Oracle (164 tables: DABL, IRIS, blpermit/gprocess) and SQL Server (10 tables: RDM, quest, autoaudit) raw tables will start loading in prod.
- Needs a full refresh after merge (`dbt_load_mode=full` pipeline input)? no. The first prod run loads these tables in full automatically.
- Grain, key, or column renames that affect downstream consumers (Tableau, parity models)? no
- Validation, verify-only run on development, pipeline #70045 (no loads):
  - sqlserver_extract_load #201031: connected, 10/10 tables counted
  - oracle_extract_load #201030: connected, 164/164 tables counted
  - informix_extract_load #201033: blocked (placeholder UCCX settings, Informix ODBC driver not installed on HWPOAAPP05)
  - Count differences are DEV lagging the source, expected.

## Checklist

- [x] `uv run pre-commit run` passes (file hygiene, ruff, ty, SQLFluff, dbt parse, lock guards)
- [ ] `uv run pyright` passes (Python changes; not a pre-commit hook)
- [ ] `uv run bl evaluate` reviewed (new dbt models / DAG changes)
- [ ] New or changed models, macros, and columns are documented in YAML, with tests
- [ ] Column names follow the abbreviation standards (`seed_abbreviation_standards.csv`)
- [ ] README / reference docs / onboarding decks assessed and updated where needed
- [ ] A significant architectural or tooling decision has an ADR in `docs/adr/`
