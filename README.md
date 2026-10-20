Select-String -Path .gitlab-ci.yml -Pattern 'dbt_parse:|dbt_compile:|workflow:|needs:' -Context 3,8
