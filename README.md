Close now (move to Done)

DI-4642 — Create automated pipeline that will run E&L daily

Why it can close: the daily 4 AM prod schedule has been running automatically since early November and loads every configured source.

The daily E&L pipeline is in place: a 4 AM scheduled pipeline on production runs the Oracle, SQL Server and PostgreSQL extract/loads automatically every day (Informix runs too and is expected to fail until its source setup is complete, DI-4655). Since 11/10 Oracle and SQL Server have loaded successfully each day (Oracle 163/163 tables after the BL view fix in DI-4653). Failure alerting and health reporting are tracked separately in DI-4659 / DI-4663. Closing.

DI-4654 — SQL Server extract/load to prod

Why it can close: sqlserver_extract_load has passed in the daily prod runs since 11/10.
Before you close it, open the last 3–4 scheduled prod pipelines and confirm sqlserver_extract_load is green in each one.

sqlserver_extract_load is running daily in the 4 AM prod schedule and has completed successfully on every run since 11/10 (e.g., pipeline #70387). No failures to address. Closing.

Update and keep In Progress

DI-4653 — Oracle extract/load to prod

What’s still open: the run time against the job time limit, and the iris_trade_names_mv row-count review.

Status: the Oracle E&L is running daily in prod. DABL.VW_FACT_SOT_FST_RETURNS was removed from the config (BL view; its date logic fails with ORA-01841; view SQL fix on Yves’s backlog), so the expected table count is now 163, not 164. Runs since the fix load 163/163 tables with 0 errors. Run time on 11/10 was 2h 14m.

Remaining from the description: (1) compare run time against the job time limit, (2) review the iris_trade_names_mv row-count difference. Will close once both are done.

DI-4655 — Informix (UCCX) extract/load to prod

What’s still open: it’s blocked until Ben finishes the source setup.

Still blocked on the Informix source setup (Ben). The job runs in the daily prod schedule and fails as expected until then. It’s listed in EL_EXPECTED_FAILURES so it doesn’t trigger the failure alert. Remove it from that list once the source is configured and the job passes.

DI-4659 — E&L failure notifications

What’s still open: promotion to test and production, plus Josh’s request to use the PS team’s email framework.

Merged to development (11/13): an el_health_check job now fails scheduled pipelines on unexpected E&L failures, so they’re no longer hidden as warnings, and GitLab’s failed-pipeline email fires. Expected failures are listed in EL_EXPECTED_FAILURES (currently Informix).

Next: (1) promote to test → production and confirm on the first 4 AM run; (2) per Josh, add the BL project to the PS team’s existing pipeline email notification framework. Following up with Dyanna.

DI-4662 — Env setting integrity check

What’s still open: promotion to test and production, then confirmation on a live run.

Merged to development (11/13): scripts/el_env_check.py runs at the start of every E&L job and fails fast with a clear message if the EL env file or a required setting is missing (the cause of the 11/5–11/8 ORACLE_USER failures). Values are never printed, and errors are reported by the health check.

Next: promote to test → production and confirm “EL env check passed” in each E&L job on the first 4 AM run. Will close after that.

DI-4663 — Daily E&L health summary

What’s still open: the Pages change isn’t in GitLab yet, and the summary isn’t published on the daily run.

Summary (el_health_summary.md) is produced by el_health_check on every scheduled run (merged to development 11/13).

Per Josh’s feedback, the next change publishes the summary with the GitLab Pages site (pages job writes it into public/, report-only, never fails the job). It’s committed on the working branch and will go into GitLab via MR once the GitHub → GitLab mirror sync is restored.

Note: the pages job doesn’t currently run on the daily 4 AM prod pipeline, so the Pages copy will appear on full builds. Daily runs are covered by the job artifact and the failure alert. Follow-up needed to publish it from the daily run.
