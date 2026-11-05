## What & why

Promotes `test` to `production` to bring in the extract/load scope change (fda2d1c): the ITDS and Tableau table groups are commented out in `src/extract_load/postgres/tables.yml`.

- **Tableau:** the GitLab runner server HWPOAAPP05 cannot reach the Tableau repository database (hwptableau02:8060). Every Tableau table timed out after ~2 minutes, which pushed the first prod load (pipeline #69708, job 199869) past the 6-hour job timeout.
- **ITDS:** excluded from the daily run for now. The first full load is already in prod (itds_line 52.8M, itds_event_details 26.7M, itds_line_hts_list 17.8M+ rows); those tables stay in prod.

This lets the "Production daily incremental" schedule run only the groups that work today.

## Type of change

- [ ] dbt models / macros / tests / seeds
- [x] Extract/load (`src/extract_load/`, `tables.yml`)
- [ ] CI/CD, tooling, or developer setup
- [ ] Reference Data Update
- [ ] Documentation only

## Data impact

- Models / sources affected: extract/load scope only. ITDS (`raw_itds.*`) and Tableau (`raw_tableau.*`) are no longer refreshed by the daily run; existing ITDS tables in prod are kept. No dbt models changed.
- Needs a full refresh after merge (`dbt_load_mode=full` pipeline input)? no
- Grain, key, or column renames that affect downstream consumers (Tableau, parity models)? no
- Queries run to validate (paste them, or note "run in " with row counts):

```sql
-- Validation via pipeline, not ad hoc SQL:
-- 1. Run the "Production daily incremental" schedule (extract_load_only = true)
-- 2. Confirm postgres_extract_load completes; check per-table row counts in the job log
-- 3. Monthly verify schedule (verify-only) compares source vs target counts
```

## Checklist

- [x] `uv run pre-commit run` passes (file hygiene, ruff, ty, SQLFluff, dbt parse, lock guards)
- [ ] `uv run pyright` passes (Python changes; not a pre-commit hook) (N/A: no Python changes)
- [ ] `uv run bl evaluate` reviewed (new dbt models / DAG changes) (N/A: no dbt changes)
- [ ] New or changed models, macros, and columns are documented in YAML, with tests (N/A)
- [ ] Column names follow the abbreviation standards (`seed_abbreviation_standards.csv`) (N/A)
- [ ] README / reference docs / onboarding decks assessed and updated where needed
- [ ] A significant architectural or tooling decision has an ADR in `docs/adr/` (N/A)

## Merge notes

- Merge with `[skip ci]` so the full push pipeline does not run in prod.
- After merge: run the "Production daily incremental" schedule manually, then the DEV and TEST mirror schedules.
- Follow-ups: request network access HWPOAAPP05 → hwptableau02:8060 (and Tableau repository access), then re-enable Tableau; decide when to re-enable ITDS.
