git switch pipeline-testing
git pull --ff-only origin pipeline-testing

git show HEAD:.gitlab-ci.yml | Select-String "check_postgres_test_access"
