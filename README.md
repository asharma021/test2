about that red dbt_retry job in #69536: dbt_retry just re-runs whatever failed in an earlier dbt_run or dbt_test in the same pipeline. Those didn’t run in #69536, so there was nothing to retry and it errored out. It’s allowed to fail, so it doesn’t affect anything.

Also, now that the pipeline is passing again, we can move forward with Umair’s dbt permit module.
