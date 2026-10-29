
# ==========================================
# PostgreSQL Incremental Extract/Load - DEV
# ==========================================
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


# ==========================================
# PostgreSQL Incremental Extract/Load - IVV
# ==========================================
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
