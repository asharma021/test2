& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
"===== MR !6 net change vs development (expect 2 files) ====="
git diff --stat origin/development...origin/feature/prod-el-readiness
"===== Simulated merge of MR !6 into current development ====="
$out = git merge-tree --write-tree --name-only origin/development origin/feature/prod-el-readiness 2>&1
if ($LASTEXITCODE -eq 0) { Write-Host "CLEAN - MR !6 can merge into development as-is" -ForegroundColor Green } else { Write-Host "CONFLICTS:" -ForegroundColor Red; $out | Select-Object -Skip 1 }
"===== Simulated merge of development into production (promotion preview) ====="
$out2 = git merge-tree --write-tree --name-only origin/production origin/development 2>&1
if ($LASTEXITCODE -eq 0) { "CLEAN" } else { "CONFLICTS:"; $out2 | Select-Object -Skip 1 }
}
