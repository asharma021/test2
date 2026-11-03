& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
"===== 1. MR !5 change to dbt/profiles.yml ====="
git --no-pager diff origin/development...origin/pipeline-testing -- dbt/profiles.yml
"===== 2. Commit(s) that changed it ====="
git log --format='%h %ad %an : %s' --date=short origin/development..origin/pipeline-testing -- dbt/profiles.yml
"===== 3. Seed schema config in dbt_project.yml ====="
git show origin/pipeline-testing:dbt/dbt_project.yml | Select-String -Pattern '^seeds:' -Context 0,12 | ForEach-Object { $_.Line; $_.Context.PostContext }
"===== 4. How extract_load_only / skip inputs affect dbt_compile and dbt_seed ====="
git show origin/pipeline-testing:.gitlab-ci.yml | Select-String 'extract_load_only|skip_compile|inputs\.' | ForEach-Object { "$($_.LineNumber): $($_.Line.Trim())" }
}
