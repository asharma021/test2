Promote development to test: mirror fix, key columns, CI cleanup



## What & why

Promotes `development` to `test` with the changes needed for prod-to-lower-environment mirroring and the CI/CD cleanup requested by Josh.

- **prod_mirror env file:** `prod_mirror` now uses the same extract/load env file (`EL_ENV_FILE`) as `postgres_extract_load`. Before this, it failed with "Missing required env vars: MIRROR_PG_*".
- **Key columns:** 7 myTTB tables inherited `unique_key: [id]` from the shared incremental settings but have no `id` column, so mirroring failed with `column "id" does not exist`. Each now uses its real key column:
  - message_details_id, message_thread_id, message_tracker_id, notification_id
  - bank_account_id
  - certificate_id (export_certificates), export_product_id (export_products)
- **CI/CD cleanup:** removed the temporary DEV/IVV extract/load jobs used during validation (`postgres_extract_load_dev_incremental`, `postgres_extract_load_ivv_incremental`, `postgres_extract_load_dev_verify`, `postgres_extract_load_test`, `prod_mirror_dev_verify`, `.postgres_incremental_guard`) and `tables_ivv_verify.yml`. All other jobs are unchanged.

## Type of change

- [ ] dbt models / macros / tests / seeds
- [x] Extract/load (`src/extract_load/`, `tables.yml`)
- [x] CI/CD, tooling, or developer setup
- [ ] Reference Data Update
- [ ] Documentation only

## Data impact

- Models / sources affected: none (no dbt changes). Extract/load config only: unique_key for 7 tables.
- Needs a full refresh after merge (`dbt_load_mode=full` pipeline input)? No for prod. For TEST, the first mirror run may need a one-time full refresh of tables previously loaded directly into IVV.
- Grain, key, or column renames that affect downstream consumers (Tableau, parity models)? No
- Validation (pipeline runs, no ad hoc SQL):

```sql
-- DEV mirror (prod -> DEV), job #200382: 46 of 53 tables, 5,485,758 rows
-- DEV mirror for the 7 key-column tables, job #200488: 7 of 7 tables, 0 errors
-- Development pipeline #69919 (after cleanup): passed
```

## Checklist

- [x] `uv run pre-commit run` passes (file hygiene, ruff, ty, SQLFluff, dbt parse, lock guards)
- [ ] `uv run pyright` passes (N/A: no Python changes)
- [ ] `uv run bl evaluate` reviewed (N/A: no dbt changes)
- [ ] New or changed models, macros, and columns are documented in YAML, with tests (N/A)
- [ ] Column names follow the abbreviation standards (N/A)
- [ ] README / reference docs / onboarding decks assessed and updated where needed
- [ ] A significant architectural or tooling decision has an ADR in `docs/adr/` (N/A)

## After merge

- Run the TEST mirror (prod → IVV) with `prod_mirror=true`, `extract_load_only=true`.
- Promote test → production (merge with `[skip ci]`).
