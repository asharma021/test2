Select-String -Path .gitlab-ci.yml -Pattern "DBT_TARGET|EL_ENV_FILE|prod_mirror" -Context 3,6
