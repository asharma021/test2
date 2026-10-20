Select-String -Path .gitlab-ci.yml -Pattern 'EL_ENV_FILE|DBT_PG_HOST|SRC_PG_' -Context 3,3
