
Promote EL_ENV_FILE fix for upstream extract/loads to production


## Summary

Defines EL_ENV_FILE once on the shared .el_upstream template so oracle, sqlserver, postgres and informix extract/load jobs all load the same env file. Before this, only postgres_extract_load had it, so Oracle, SQL Server and Informix failed in prod on missing INFORMIX_HOST / MSSQL_USER. Postgres behavior is unchanged.

## Type of change

- [ ] dbt models / macros / tests / seeds
- [x] Extract/load (`src/extract_load/`, `tables.yml`)
- [x] CI/CD, tooling, or developer setup
- [ ] Reference Data Update
- [ ] Documentation only

## Data impact

- Models / sources affected: No model changes. Oracle (164 tables: DABL, IRIS, blpermit/gprocess) and SQL Server (10 tables: RDM, quest, autoaudit) raw tables will start loading in prod from the next 4 AM build.
- Needs a full refresh after merge (`dbt_load_mode=full` pipeline input)? no. First prod run loads these tables in full automatically.
- Grain, key, or column renames that affect downstream consumers (Tableau, parity models)? no
- Validation, verify-only run on development, pipeline #70045 (no loads):
  - sqlserver_extract_load #201031: connected, 10/10 tables counted
  - oracle_extract_load #201030: connected, 164/164 tables counted
  - informix_extract_load #201033: blocked (UCCX settings and Informix ODBC driver pending, Ben / PS)
- MR pipeline #70062 passed after moving EL_ENV_FILE to .el_upstream.

## Checklist

- [x] `uv run pre-commit run` passes (file hygiene, ruff, ty, SQLFluff, dbt parse, lock guards)
- [ ] `uv run pyright` passes (Python changes; not a pre-commit hook)
- [ ] `uv run bl evaluate` reviewed (new dbt models / DAG changes)
- [ ] New or changed models, macros, and columns are documented in YAML, with tests
- [ ] Column names follow the abbreviation standards (`seed_abbreviation_standards.csv`)
- [ ] README / reference docs / onboarding decks assessed and updated where needed
- [ ] A significant architectural or tooling decision has an ADR in `docs/adr/`
