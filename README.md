
Select-String -Path .gitlab-ci.yml `
  -Pattern '^\.el_job:', '^\.el_upstream:', '^postgres_extract_load:', '^postgres_extract_load_test:' `
  -Context 2,35
