postgres_extract_load:
  extends: .el_upstream
  variables:
    EL_MODULE: "postgres"
    EL_LABEL: "PostgreSQL Extract/Load"
    EL_ENV_FILE: '\\thqpcelerra01\ttb_common\_OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'


    
