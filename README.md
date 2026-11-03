& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null

"===== 1. Local state (expect pipeline-testing, no changes listed) ====="
git rev-parse --abbrev-ref HEAD
git status --short --untracked-files=no

"===== 2. Branch tips ====="
foreach ($b in 'production','test','development','pipeline-testing','feature/prod-el-readiness') {
  $t = git log -1 --format='%h %ad %s' --date=short "origin/$b" 2>$null
  "{0,-28} {1}" -f $b, $(if ($t) { $t } else { '(branch not found)' })
}

"===== 3. MR !16: commits not in development (expect only aa1ac56) ====="
git log --oneline origin/development..origin/feature/prod-el-readiness

"===== 4. MR !16: files changed (expect 2) ====="
git diff --stat origin/development...origin/feature/prod-el-readiness

"===== 5. Does MR !16 depend on pipeline-testing? ====="
git merge-base --is-ancestor origin/feature/prod-el-readiness~1 origin/development
"MR !16 parent is already in development (no dependency on pipeline-testing): $($LASTEXITCODE -eq 0)"
git merge-base --is-ancestor origin/feature/prod-el-readiness origin/pipeline-testing
"aa1ac56 already inside pipeline-testing (expect False): $($LASTEXITCODE -eq 0)"

"===== 6. MR !15: commits on pipeline-testing not in development ====="
"count: $(git rev-list --count origin/development..origin/pipeline-testing)   tip: $(git log -1 --format='%h' origin/pipeline-testing) (expect 871f7a9)"

"===== 7. Overlap: does MR !15 touch the lines MR !16 changes? (expect nothing listed) ====="
git diff origin/development...origin/pipeline-testing -- dbt/profiles.yml | Select-String 'TODO-prod-host|dbname:\s*business_layer_prod'
git diff origin/development...origin/pipeline-testing -- .gitlab-ci.yml | Select-String 'CI_DEFAULT_BRANCH'

"===== 8. Simulated merge of MR !15 + MR !16 together (no files touched) ====="
$out = git merge-tree --write-tree --name-only origin/pipeline-testing origin/feature/prod-el-readiness 2>&1
if ($LASTEXITCODE -eq 0) { "CLEAN - MR !15 and MR !16 are independent; either can merge first" } else { "CONFLICTS:"; $out | Select-Object -Skip 1 }
}
