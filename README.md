Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>&1 | Out-Null
git log --oneline -3 origin/pipeline-testing
"Removed-job count on origin: " + @(git show origin/pipeline-testing:.gitlab-ci.yml | Select-String -Pattern '^(postgres_dev_target_inspect|postgres_ivv_target_inspect|mirror_coverage_inspect|check_postgres_test_access):').Count
