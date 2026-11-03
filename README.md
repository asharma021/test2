Hey Yves, update:

The intermittent dbt_evaluate failure is fixed (MR !19). The evaluator itself always passed; the job was failing because of how the Windows PowerShell runner handled warning output from that step, so the Code Quality report step never ran. Now it runs and the job passes.
The development pipeline (#69590) and the MR !17 pipeline (#69591) are both fully green: env_setup, dbt_compile, dbt_evaluate and all quality checks.
MR !17 (development to test) is ready for your approval. On merge, the test pipeline will run dbt seed/run/test into IVV (business_layer_test).
Prod: MR !6 is merged into development. Still waiting on Ben for the prod write login, then we promote to production, run the first prod Postgres load and validate mirroring into DEV.
