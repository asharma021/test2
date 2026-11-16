DI-4642 — Create automated pipeline that will run E&L daily → CLOSE

The daily E&L pipeline is in place. A 4 AM scheduled pipeline on production runs the Oracle, SQL Server and PostgreSQL extract/loads automatically every day (Informix also runs and is expected to fail until its source is set up, see DI-4655). Oracle and SQL Server have loaded successfully on the daily runs since 11/10. Failure alerting and the health summary are tracked in DI-4659 and DI-4663. Closing.

DI-4654 — SQL Server extract/load to prod → CLOSE

sqlserver_extract_load runs daily in the 4 AM prod schedule and has completed successfully on each run since 11/10 (e.g., pipeline #70387). No open issues. Closing.

(Before closing, confirm the last few scheduled runs show sqlserver_extract_load green.)

DI-4653 — Oracle extract/load to prod → IN PROGRESS

Oracle E&L runs daily in prod. DABL.VW_FACT_SOT_FST_RETURNS was removed from the config (BL view whose date logic fails with ORA-01841; view SQL fix is on the backlog), so the expected table count is now 163, not 164. Runs since the fix load 163/163 tables with 0 errors. Run time on 11/10 was 2h 14m. Remaining: compare run time to the job time limit and review the iris_trade_names_mv row-count difference.

DI-4655 — Informix (UCCX) extract/load to prod → IN PROGRESS (blocked)

Blocked on the Informix source setup (Ben). The job runs in the daily prod schedule and fails as expected until then. It’s listed in EL_EXPECTED_FAILURES, so it doesn’t trigger the failure alert. Will remove it from that list and close once the source is configured and the job passes.

DI-4659 — E/L failure notifications → IN PROGRESS

Merged to development on 11/13. The new el_health_check job fails scheduled pipelines on unexpected E&L failures, so they’re no longer hidden as warnings, and GitLab’s failed-pipeline email is sent. Next: promote to test and production and confirm on the first 4 AM run. Per Josh, also add the BL project to the PS team’s existing pipeline email notification framework (following up with Dyanna).

DI-4662 — Env setting integrity check → IN PROGRESS

Merged to development on 11/13. scripts/el_env_check.py runs at the start of every E&L job and fails fast with a clear message if the EL env file or a required setting is missing (the cause of the 11/5–11/8 ORACLE_USER failures). Values are never printed. Next: promote to test and production, and confirm “EL env check passed” on the first 4 AM run before closing.

DI-4663 — Daily E/L health summary → IN PROGRESS

The summary (el_health_summary.md) is produced by el_health_check on every scheduled run (merged to development on 11/13). Per Josh’s feedback, a follow-up change publishes it with the GitLab Pages site. It’s committed on the working branch and goes into GitLab by MR once the GitHub → GitLab sync is restored. Note: pages doesn’t currently run on the daily 4 AM pipeline, so publishing it from the daily run needs a follow-up.
