git switch development
git pull --ff-only origin development
git status --short


git switch -c feature/postgres-el-test-validation


Select-String -Path .gitlab-ci.yml -Pattern 'postgres_extract_load:','DBT_TARGET:','EL_ENV_FILE:','EL_VERIFY_ONLY:','pipeline-testing','development' -Context 3,12
