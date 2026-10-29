Select-String -Path .gitlab-ci.yml `
  -Pattern '^\.postgres_incremental_guard:', '^postgres_extract_load_dev_incremental:', '^postgres_extract_load_ivv_incremental:', '^postgres_extract_load_test:' `
  -Context 0,75
