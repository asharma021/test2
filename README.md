& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
"===== 1. The test (origin/development) ====="
git show origin/development:dbt/tests/generic/no_deprecated_abbreviations.sql
"===== 2. Where it is attached ====="
git show origin/development:dbt/tests/generic/_generic.yml
"===== 3. Other run_query / execute guards in the project (existing convention) ====="
git grep -n -E 'flags\.WHICH|invocation_args_dict|if execute' origin/development -- dbt/macros dbt/tests | Select-Object -First 20
"===== 4. Other jobs that run dbt compile / docs generate ====="
git show origin/development:.gitlab-ci.yml | Select-String 'dbt compile|dbt docs generate|dbt build' | ForEach-Object { "$($_.LineNumber): $($_.Line.Trim())" }
}
