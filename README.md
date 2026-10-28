variables:
  EL_MODULE: "postgres"
  EL_LABEL: "PostgreSQL IVV Verify-Only"
  EL_CONFIG: "src/extract_load/postgres/tables_iw_verify.yml"
  EL_ENV_FILE: '\\tthqpcelerra01\ttb_common\OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'
  DBT_TARGET: "test"
  EL_VERIFY_ONLY: "true"
  DBT_SELECT: "myttb_cg_ref_identifier_type"
  DBT_LOAD_MODE: "incremental"
