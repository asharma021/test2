Hi Josh, quick update on the PostgreSQL Extract/Load work on pipeline-testing:

• DEV: incremental load and rerun validated on raw_myttb.myttb_controlled_group – source/target counts match (2,982 = 2,982) after each run, 0 duplicate IDs.
• IVV: first load into the IVV target plus rerun validated – 12 = 12 against the IVV target.
• Mirroring: CI setup is ready (credentials, prod read-only connection, single-table selection all verified).

All jobs are manual and limited to pipeline-testing; no production changes. Evidence is in DI-4642, and I've opened an MR to development for your review.

Three decisions before I take the next step:
1. Mirroring – the prod source (oa-prod-flexible-server / oablmsdb) currently has no raw_* tables. Should prod raw data be in oablmsdb, or in business_layer_prod as listed in profiles.yml?
2. IVV incremental – should I add an IVV-source incremental table (e.g., controlled_group) to tables_ivv_verify.yml, or load IVV from tables.yml (prod source) the same way as DEV?
3. The myTTB incremental tables use delete_window, which can leave duplicate IDs if an already-loaded row is updated. Keep it, or switch those tables to upsert?
