
Promote development to test: exclude source: DEV groups from prod Postgres E&L; Oracle target settings (DI-4665)


## What & why

Promotes the latest development changes to test:

1. **Exclude the `source: DEV` groups from prod Postgres E&L.** `postgres_extract_load` now also
   excludes `tag:itds tag:cbma_claims tag:foreignproducer` (3 groups, 57 tables) so DEV-source
   tables don't load into prod until confirmed. The prod foreignproducer group (no tag) still
   loads. The 3 workflows792026 tables stay excluded.
2. **Oracle target settings (DI-4665).** `oracle_target_load` sets `EL_ORACLE_TEXT_TYPE='VARCHAR2(4000 CHAR)'`
   and `ORACLE_CLIENT_BIN_DIR` (Oracle client on python-app05, thick mode). The job stays
   development-only and inactive unless `EL_ORACLE_TARGET_ENABLED=true`; it is never created on test or prod.

Jira: DI-4665

## Type of change

- [ ] dbt models / macros / tests / seeds
- [ ] Extract/load (`src/extract_load/`, `tables.yml`)
- [x] CI/CD, tooling, or developer setup
- [ ] Reference Data Update
- [ ] Documentation only

## Data impact

- Models / sources affected: none in test. Upstream E&L runs only on production; there, the
  3 `source: DEV` groups and the 3 workflows792026 tables are skipped.
- Needs a full refresh after merge (`dbt_load_mode=full` pipeline input)? no
- Grain, key, or column renames that affect downstream consumers (Tableau, parity models)? no
- Queries run to validate: none (CI only). Oracle target job tested on development (job #208572):
  Oracle connection and thick mode work; source settings pending.

## Checklist

- [ ] `uv run pre-commit run` passes: not run (CI-only YAML change; YAML validated)
- [ ] `uv run pyright` passes: N/A, no Python changes
- [ ] `uv run bl evaluate` reviewed: N/A, no dbt model changes
- [ ] New or changed models, macros, and columns are documented in YAML, with tests: N/A
- [ ] Column names follow the abbreviation standards: N/A
- [ ] README / reference docs / onboarding decks assessed and updated where needed: N/A
- [ ] A significant architectural or tooling decision has an ADR in `docs/adr/`: N/A
