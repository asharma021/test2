Prod extract/load readiness (2026-11-02)

Following Josh’s direction, DEV/IVV extract/load testing is complete, and the focus is now prod extract/load and then mirroring validation.
Branch review (read-only): production (default branch) still targets DEV and has never loaded prod. Promoting development → production merges cleanly except for uv.lock.
Draft MR !6 (feature/prod-el-readiness → development):
prod target fixed (oablmsdb, schema business_layer_prod);
prod extract/load made manual and limited to a single table, so it can’t run unscoped automatically.
Fixed a corrupted local Python environment (bad uv cache entries); the same issue likely caused the MR !15 env_setup failure. Runner fix in progress.
Waiting on: PS for prod host confirmation and a write login; Josh’s review of MR !6.
Next: development → test → production promotion, a single-table prod load with verify, then mirror verify and a scoped mirror run into DEV.
