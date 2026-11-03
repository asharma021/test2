& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
"===== MR !6 net change vs development (expect 2 files) ====="
git diff --stat origin/development...origin/feature/prod-el-readiness
"===== MR !6 into current development ====="
git merge-tree --write-tree --name-only origin/development origin/feature/prod-el-readiness 2>&1 | Out-Null
if ($LASTEXITCODE -eq 0) { Write-Host "CLEAN - MR !6 merges as-is" -ForegroundColor Green } else { Write-Host "CONFLICTS - send me this output" -ForegroundColor Red; git merge-tree --write-tree --name-only origin/development origin/feature/prod-el-readiness | Select-Object -Skip 1 }
"===== Promotion preview: development into production ====="
$o = git merge-tree --write-tree --name-only origin/production origin/development 2>&1
if ($LASTEXITCODE -eq 0) { "CLEAN" } else { "CONFLICTS:"; $o | Select-Object -Skip 1 }
"===== Promotion preview: development into test ====="
$o2 = git merge-tree --write-tree --name-only origin/test origin/development 2>&1
if ($LASTEXITCODE -eq 0) { "CLEAN" } else { "CONFLICTS:"; $o2 | Select-Object -Skip 1 }
}
