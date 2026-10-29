
# PostgreSQL incremental loading - DEV
postgres_extract_load_dev_incremental:
  extends: .el_job
  stage: extract_load
  environment:
    name: dev
  tags:
    - python-app05
  variables:
    EL_MODULE: "postgres"
    EL_LABEL: "PostgreSQL DEV Incremental Load"
    DBT_TARGET: "dev"
    EL_VERIFY_ONLY: "false"
    DBT_LOAD_MODE: "incremental"
  resource_group: postgres-el-dev
  allow_failure: false
  rules:
    - when: never

# PostgreSQL incremental loading - IVV
postgres_extract_load_ivv_incremental:
  extends: .el_job
  stage: extract_load
  environment:
    name: ivv
  tags:
    - python-app05
  variables:
    EL_MODULE: "postgres"
    EL_LABEL: "PostgreSQL IVV Incremental Load"
    DBT_TARGET: "test"
    EL_VERIFY_ONLY: "false"
    DBT_LOAD_MODE: "incremental"
  resource_group: postgres-el-ivv
  allow_failure: false
  rules:
    - when: never
















    
Get-ChildItem src/extract_load -Recurse -Filter "*.py" |
  Select-String -Pattern "ensure_state_table|DBT_LOAD_MODE|incremental|upsert|checkpoint|watermark" |
  Select-Object Path, LineNumber, Line

