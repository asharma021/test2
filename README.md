
Select-String -Path .gitlab-ci.yml `
  -Pattern '^postgres_extract_load:', '^postgres_extract_load_test:', '^postgres_extract_load_dev_incremental:', '^postgres_extract_load_ivv_incremental:' `
  -Context 2,3
