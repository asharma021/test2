Select-String -Path .gitlab-ci.yml -Pattern '^\s*---\s*$' |
Select-Object LineNumber, Line


Select-String -Path .gitlab-ci.yml `
-Pattern '^\.postgres_incremental_guard:', '^postgres_extract_load_dev_incremental:', '^postgres_extract_load_ivv_incremental:' `
-Context 0,8
