Prepares the prod PostgreSQL extract/load per Josh’s direction (run extract/load on the prod server, then validate mirroring).

Prod target: dbname → oablmsdb; business_layer_prod stays the schema. The host placeholder is replaced with oa-prod-flexible-server (to be confirmed by PS).
The prod extract/load on production is now manual and defaults to a single table (myttb_cg_ref_identifier_type, 12 rows), so promotion can’t start an automatic, unscoped prod load.
Validated locally: YAML parse, all pre-commit hooks including dbt-parse.
Not included: prod credentials (PS), promotion to test/production, mirror run.
