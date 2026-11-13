DI-4659 — E&L failure notifications

Implemented in MR (branch feature/el-failure-notifications, target development).

Each E&L job now records its final status (logs/<job>.status).
New el_health_check job runs at the end of scheduled/web pipelines, reads each E&L job’s log and status, and fails the pipeline when an E&L job or table fails unexpectedly. Failures are no longer hidden as yellow warnings, and GitLab sends its standard “Pipeline failed” email.
Known failures are listed in EL_EXPECTED_FAILURES (currently Informix, until its source is set up).
Tested locally against 4 scenarios.

Alert routing: the email goes to the schedule owner by default. To send it to the team/PS, enable Settings → Integrations → Pipeline status emails (broken pipelines only, default branch).

Next: review/merge, promote, confirm on the first 4 AM prod run.

DI-4663 — Daily E&L health summary

Delivered with DI-4659 in the same MR. el_health_check writes el_health_summary.md on every scheduled run: job, result, tables processed, rows loaded and failed tables, plus the error lines for any failed job. Available as a job artifact (30 days) and printed in the job log.

Next: review/merge, promote, confirm on the first 4 AM prod run.

DI-4662 — Env setting integrity check

Implemented in MR (branch feature/el-failure-notifications, target development).

New scripts/el_env_check.py runs at the start of every E&L job, before any data moves. It loads the EL env file the same way the loaders do and fails fast with a clear message if the file isn’t usable, a required setting is missing or empty (e.g., ORACLE_USER, which caused the 11/5–11/8 Oracle failures), or the target login/endpoint can’t be resolved.
Values are never printed. Errors go to the job log, so the health check reports them in the alert.
EL_ENV_CHECK=off bypasses the check if ever needed.
Tested locally against 6 scenarios, including confirming no secrets appear in output.

Next: review/merge, promote, confirm on the first 4 AM prod run.

DI-4653 — Oracle extract/load to prod

The production fix (VW_FACT_SOT_FST_RETURNS commented out of the Oracle E&L config) is now also in the development MR (branch feature/el-failure-notifications), so development and test match production. The view SQL fix remains on the backlog.
