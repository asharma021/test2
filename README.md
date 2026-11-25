DI-4659: In Progress

Oracle, SQL Server and Postgres E&L now fail the pipeline on real errors (in production since 11/20); Informix remains an allowed warning while setup is pending. Remaining: onboarding the Business Layer pipeline to the PS notification framework.

DI-4665: In Progress

Added CI job oracle_target_load: runs extract-load-postgres --target-db oracle --select tag:legacy_oracle after the Postgres E&L. It is created only on the development branch when EL_ORACLE_TARGET_ENABLED=true and never runs on test or prod. Inactive as requested. Before enabling in dev: SRC_PG_BUSINESS_LAYER settings in the EL env file, dev TARGET_ORA variables, and Yves's go-ahead.

DI-4668: In Progress

First draft written (docs/runbooks/el-operations.md): what runs, reading failures, env pre-flight, common failures, reruns and recovery, monthly audit, Oracle target load. Next: update for the IVV/DEV source groups and the current daily-run scope, add contacts, then review with Josh and PS.
