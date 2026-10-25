git switch -c feature/postgres-el-test-validation
git branch --show-current


Get-Content .gitlab-ci.yml | Select-Object -Skip 540 -First 45
