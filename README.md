PostgreSQL E/L – DEV and IVV validation complete (pipeline-testing); merge request opened to development for review.

DEV (prod myTTB → DEV OA): incremental run + rerun on raw_myttb.myttb_controlled_group (jobs 197624, 197629), verify 2,982 = 2,982 after each (197625, 197630), 0 duplicate ids. Approved DEV verify 12 = 12 (197696).
IVV (IVV myTTB → IVV OA): first load into empty IVV target, rerun, verify 12 = 12 (197673). Correction: the earlier "IVV 12/12" compared IVV source with the DEV target.
Mirroring: CI setup ready (credentials, prod read-only connection, single-table selection verified); blocked because the prod source has 0 raw_* tables.
Config fixes: env-file folder (\_OA_Data_Sharing\), IVV env file and mapping-path typo; temporary diagnostic jobs removed. All jobs manual on protected pipeline-testing; no production changes.

Questions:
1. Mirror source: is prod raw data expected in oablmsdb or business_layer_prod (profiles.yml prod host still TODO)?
2. IVV incremental: add an IVV-source incremental entry to tables_ivv_verify.yml, or use tables.yml (prod source) like DEV?
3. Keep delete_window for the myTTB incremental tables or switch to upsert (delete_window can leave duplicate ids when rows are updated)?
