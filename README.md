& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
"===== 1. Seed file and when it was added ====="
git ls-tree -r --name-only origin/pipeline-testing | Select-String 'seed_abbreviation_standards' | ForEach-Object { $_.Line }
git log origin/pipeline-testing --diff-filter=A --format='%h %ad %an : %s' --date=short -- '*seed_abbreviation_standards*'
"===== 2. Same seed on development? ====="
git ls-tree -r --name-only origin/development | Select-String 'seed_abbreviation_standards' | ForEach-Object { $_.Line }
"===== 3. Where it is used ====="
git grep -n 'seed_abbreviation_standards' origin/pipeline-testing -- dbt | Select-Object -First 15
"===== 4. Does MR !5 change anything under dbt/ ? (expect only profiles.yml or nothing) ====="
git diff --stat origin/development...origin/pipeline-testing -- dbt
"===== 5. dbt_seed job definition on pipeline-testing ====="
git show origin/pipeline-testing:.gitlab-ci.yml | Select-String -Pattern '^dbt_seed:' -Context 0,30 | ForEach-Object { $_.Line; $_.Context.PostContext }
}
