PostgreSQL E/L – DEV and IVV validation (pipeline-testing)

Fixes on pipeline-testing (all manual jobs, no production changes): corrected the env-file folder (\_OA_Data_Sharing\); pointed the IVV verify job at extract_load.env; fixed a typo in the IVV mapping path (tables_iv_verify → tables_ivv_verify); configured the DEV and IVV load jobs (single table, manual, guard unchanged); added temporary read-only inspection jobs.

DEV (source prod myTTB myttbmsdb, read-only → target DEV oablmsdb), table raw_myttb.myttb_controlled_group:

Incremental run (job 197624) and rerun (job 197629): same window, 2 rows cleared and reloaded each time, 0 errors.
Verify after each run (jobs 197625, 197630): 2,982 = 2,982, match.
Read-only inspection: 2,982 rows, 2,982 distinct IDs, 0 duplicates; watermark unchanged; only the selected table touched.

IVV (source IVV myTTB test-flexible-server-customapps, target IVV oa-tst-flexible-server/oablmsdb), table raw_myttb.myttb_cg_ref_identifier_type:

The IVV target was empty (no raw_myttb schema). The first load created the schema and table, 12 rows, 0 errors. The rerun reloaded the same 12 rows.
Verify after the load (job 197673) and after the rerun: 12 = 12, match, the first verification against the IVV target.
Correction: the earlier “IVV 12/12” compared the IVV source with the DEV target, not the IVV target.

Open items:

an IVV incremental test (needs a table with incremental config from the IVV source);
PostgreSQL mirroring;
the delete_window strategy can leave duplicate IDs when already-loaded rows are updated (upsert exists as an option);
the approved postgres_extract_load block still uses the old folder path;
the prod host in profiles.yml is still a TODO;
an incremental load of the DEV table on 10/30 16:44 UTC came from outside these jobs;
cleanup before the merge request.
