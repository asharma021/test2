Validation

Changes: 2 files.
dbt/profiles.yml prod target: host oa-prod-flexible-server, dbname oablmsdb, schema business_layer_prod.
postgres_extract_load on production: manual, with DBT_SELECT=myttb_cg_ref_identifier_type.
Pre-commit hooks passed; dbt parse --target prod passed.
Pipeline editor simulation on production: postgres_extract_load shows as manual.
Merges cleanly into the current development.
Promotion preview: development → test is clean; development → production conflicts only in uv.lock, which will be resolved by keeping development’s version.

Note for promotion: only Postgres is manual on production. The Oracle, SQL Server and Informix loads and the dbt jobs run automatically on the first production pipeline, as currently designed.

Pending: prod write login from PS (DBT_PG_USER / DBT_PG_PASSWORD scoped to prod).
