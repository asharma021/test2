## Summary
Validates PostgreSQL Extract/Load end-to-end in DEV and IVV using the existing E/L code (no Python changes). All new jobs are **manual** and restricted to web pipelines on protected `pipeline-testing`; no job runs automatically and nothing touches production.

## Changes (.gitlab-ci.yml only, plus earlier IVV commits on this branch)
- `.el_job`: pass `--config` when `EL_CONFIG` is set.
- `postgres_extract_load`: identical to the approved `development` block.
- `.postgres_incremental_guard` + `postgres_extract_load_dev_incremental` / `postgres_extract_load_ivv_incremental`: manual single-table loads (DEV: `tables.yml`, `myttb_controlled_group`; IVV: `tables_ivv_verify.yml`, source profile IVV, `myttb_cg_ref_identifier_type`).
- `postgres_extract_load_dev_verify`: manual verify-only for the DEV test table.
- `postgres_extract_load_test`: IVV verify-only; fixed env-file folder (`\_OA_Data_Sharing\`), env file (`extract_load.env`) and mapping path typo (`tables_iv_verify` → `tables_ivv_verify`).
- `prod_mirror_dev_verify`: manual mirror readiness check (verify-only); `prod_mirror` itself unchanged.
- Temporary diagnostic jobs added during testing were removed (`871f7a9`).
- Note: this branch also carries the earlier `dbt/profiles.yml` TEST profile change (`a2d1ead`).

## Evidence
**DEV** (source prod myTTB `myttbmsdb` read-only → target DEV `oablmsdb`), `raw_myttb.myttb_controlled_group`, strategy `delete_window`:
| Step | Job | Result |
|---|---|---|
| Baseline verify | 197615 | 2,982 = 2,982 |
| Incremental run 1 | 197624 | window `>= 2026-08-16 14:27:10.975904`, 2 cleared / 2 loaded, 0 errors |
| Verify | 197625 | 2,982 = 2,982 |
| Incremental rerun | 197629 | same window, 2 cleared / 2 loaded, 0 errors |
| Verify | 197630 | 2,982 = 2,982 |
| Read-only inspection (before/after both runs) | — | 2,982 rows, 2,982 distinct ids, **0 duplicates**; watermark unchanged; other tables untouched |
| Approved DEV verify after cleanup | 197696 | `myttb_cg_ref_identifier_type` 12 = 12 |

**IVV** (source IVV myTTB `test-flexible-server-customapps` → target IVV `oa-tst-flexible-server/oablmsdb`), `raw_myttb.myttb_cg_ref_identifier_type`:
| Step | Job | Result |
|---|---|---|
| Verify before load | 197643 | target absent (IVV target was empty) |
| First load | (pipeline on 54d34eb) | schema + table created, 12 rows, 0 errors |
| Verify | 197673 | **12 = 12** |
| Rerun (full reload) + verify | (same pipeline) | 12 rows, 12 = 12 |

**Mirror readiness:** all `MIRROR_PG_*` settings present; prod read-only connection (`oa-prod-flexible-server/oablmsdb`, `prd_dabl_ro_srvc_acct`) works; single-table selection honored. Catalog check: **0 `raw_*` tables in the prod source** (DEV has 355), so there is nothing to mirror yet.

## Known limitations / open items
- `delete_window` (default for the myTTB incremental mappings) is documented in the code as correct only when the cursor never moves; updated rows can leave duplicate ids. `upsert` exists as an option.
- Earlier "IVV 12/12" compared IVV source with the DEV target; the IVV-target results above supersede it.
- Runner `.venv` shows a recurring `zipp` RECORD warning (harmless; one-time cleanup recommended).

## Decisions needed (Josh)
1. **Mirror source:** prod raw data — `oablmsdb` (current `MIRROR_PG_DATABASE`) or `business_layer_prod` (`profiles.yml`, host still TODO)?
2. **IVV incremental:** add an IVV-source incremental entry (e.g. `controlled_group`) to `tables_ivv_verify.yml`, or load IVV from `tables.yml` (prod source) like DEV?
3. **Strategy:** keep `delete_window` for the myTTB tables or move to `upsert`?
4. FYI: an incremental load of `raw_myttb.myttb_controlled_group` into DEV ran on 2026-10-30 16:44 UTC outside these jobs.
