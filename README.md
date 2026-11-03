Promotes development to test. The latest development pipeline is fully green.

Includes:

the dbt_compile deadlock fix;
CI fixes (no stuck pipelines, env_setup runs automatically after merges);
Umair’s IVV extract/load config;
Srini’s DI-4645 permit module changes;
prod-readiness config (MR !6), if merged.

test is still on the old design, so this brings the new pipeline to test. On merge, the test pipeline runs dbt seed/run/test into IVV (business_layer_test). Merges cleanly.
