Promote development to test: fail pipeline on E&L failures, remove el_health_check (DI-4659)




## What & why

Promotes development to test. Main change (DI-4659 rework, per team review on 11/18):

- `oracle_extract_load`, `sqlserver_extract_load`, `postgres_extract_load`: `allow_failure: false`. A failed load (any failed table) now fails the pipeline and stops the downstream dbt jobs.
- `informix_extract_load` and `prod_mirror` unchanged (still allowed to fail; Informix setup pending).
- Removed the `el_health_check` job and `scripts/el_health_check.py`, its `pages` lines, and the `.el_job` status after_script and artifact.
- `pages_audit` rules use `when: always`, so the monthly verify-only audit page still publishes when it finds mismatches.
- Env pre-flight check (`el_env_check.py`, DI-4662) unchanged.

Also included from development (already reviewed and merged there): DI-4644 (reference data seed update) and DI-4671 (`REF_SUBMSN_WRKFLW_STATUS` model).

## Type of change

- [x] dbt models / macros / tests / seeds (DI-4644, DI-4671)
- [ ] Extract/load (`src/extract_load/`, `tables.yml`)
- [x] CI/CD, tooling, or developer setup
- [x] Reference Data Update (DI-4644)
- [ ] Documentation only

## Data impact

- CI change: no data impact. The upstream E&L jobs run only on production, so on test this affects only how failures are reported.
- Needs a full refresh after merge? **No**
- Grain, key, or column renames? See DI-4671 for model details.
- Validation: development pipeline after the merge of MR !40 passed. CI YAML parsed before/after; only `.el_job`, `pages`, `pages_audit` and the three E&L jobs changed, with no remaining references to `el_health_check`.

## Note before promoting to production

Once this reaches production, Postgres will fail the 4 AM run while the 3 `myttb_workflows792026_*` tables (missing source schema, from the IVV change) are still in `tables.yml`. Those need to be fixed by the owner before the test → production promotion.

Refs: DI-4659, DI-4644, DI-4671
