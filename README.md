PostgreSQL E/L – DEV incremental validation complete (pipeline-testing, pipeline #69094)

Fixed the DEV env-file path: the share folder is \_OA_Data_Sharing\ and the job pointed to \OA_Data_Sharing\. The runner now loads extract_load.env.
Confirmed connections from job logs: source = myTTB prod (myttbmsdb, read-only), target = DEV OA (oablmsdb).
Test table raw_myttb.myttb_controlled_group (cursor last_modified_date, key id, fuzz 2 days, strategy delete_window).
Incremental run 1 (job 197624) and rerun (job 197629): same window, 2 rows cleared and reloaded each time, 0 errors.
Built-in verify after each run (jobs 197625, 197630): source 2,982 = target 2,982, match.
Read-only inspection job: 2,982 rows, 2,982 distinct IDs, 0 duplicates after both runs. Watermark unchanged, state row updated, other tables untouched.
All jobs are manual and limited to protected pipeline-testing; no production writes.
Open items: the delete_window strategy can leave duplicate IDs when already-loaded rows are updated (documented in the code; upsert exists as an option); the approved postgres_extract_load and the IVV verify job still use the old folder path; the prod host in profiles.yml is still a TODO.
Next: IVV verify, then IVV incremental.
