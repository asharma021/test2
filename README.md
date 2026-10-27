Select-String -Path .gitlab-ci.yml -Pattern 'SRC_PG_|DBT_TARGET|EL_ENV_FILE' -Context 2,3
