
Select-String -Path .gitlab-ci.yml `
  -Pattern 'EL_ENV_FILE:', 'EL_CONFIG:', 'DBT_TARGET:', 'environment:' `
  -Context 2,3






Select-String -Path .gitlab-ci.yml `
  -Pattern '^postgres_extract_load_dev_incremental:', '^postgres_extract_load_ivv_incremental:' `
  -Context 0,20
