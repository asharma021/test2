Select-String -Path .gitlab-ci.yml -Pattern 'extract_load_pg|postgres|EL_MODULE|EL_ENV_FILE|EL_SELECT|EL_FULL_REFRESH' -Context 8,15
