2026-11-03 update

Runner environment: env_setup failure (pydantic-core mismatch) resolved by clearing runner caches and rebuilding the environment.
dbt_compile deadlock: the compile step failed with business_layer_seed.seed_abbreviation_standards does not exist. The no_deprecated_abbreviations test queried the seed during compile, which runs before dbt_seed.
Fixed in MR !9 (development); dbt_compile passed in pipeline #69536.
Same fix applied to remote-development.
Also prevents this failure on the first IVV and prod runs.
Stuck pipelines: MR !13 merged. Unprotected branches no longer create pipelines (the protected runner can’t run them); pipelines run after merge into protected branches.
DEV/IVV extract/load: MR !5 merged into development.
Prod readiness (MR !6): prod target fixed and postgres_extract_load manual on production with a single-table default.
Validated with dbt parse --target prod and a Pipeline editor simulation.
Merges cleanly into development. Out for review.
Waiting on:
PS (Ben): prod write login (DBT_PG_USER / DBT_PG_PASSWORD, prod scope). Host reachability is already confirmed via the read-only mirror check.
Yves: MR !6 review and promotion approval.
Next:
Promote development → test → production.
First prod Postgres load: verify-only → load (12 rows) → verify.
Mirroring validation into DEV.
