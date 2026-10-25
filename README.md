git show origin/pipeline-testing:.gitlab-ci.yml | Select-String "check_postgres_test_access" -Context 0,12
