Select-String -Path .gitlab-ci.yml -Pattern 'EL_ENV_FILE|DBT_TARGET: "test"|postgres_extract_load:' -Context 2,3
