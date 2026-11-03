Hey Yves, that’s done. With Umair’s and Srini’s changes both in development, the latest development pipeline (#69572) is fully green, including dbt_compile and dbt_evaluate. Quick summary of what changed today:

dbt_compile failures: fixed (MR !9). The seed test no longer queries the database during compile.
Stuck pipelines: fixed (MR !13). Unprotected branches don’t create pipelines anymore; pipelines run once the MR is merged into development.
“Blocked” after merges: fixed (MR !16). env_setup and dbt_compile now run automatically after each merge, so approved MRs get checked right away with no manual click.
Old feature branches still have the old CI file, so they’ll show stuck pipelines until they merge in the latest development.
Since Srini’s work is in development now, I’ll close the remote-development fix MR, as it isn’t needed anymore.
Only MR !6 (prod readiness) is left. It’s waiting on your review and on Ben for the prod write login.
