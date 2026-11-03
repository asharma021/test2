& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
"===== 1. Commits on development not yet on test ====="
git log --oneline --no-merges origin/test..origin/development | Select-Object -First 40
"===== 2. Umair's commits on development ====="
git log --format='%h %ad %an : %s' --date=short origin/development --author='Umair' -10
git log --format='%h %ad %an : %s' --date=short origin/development --author='Iqbal' -10
"===== 3. Files changed development vs test ====="
git diff --stat origin/test origin/development | Select-Object -Last 25
"===== 4. Extract/load config changes vs test (IVV incremental) ====="
git --no-pager diff origin/test origin/development -- src/extract_load/postgres/tables.yml | Select-Object -First 80
"===== 5. IVV verify mapping untouched? ====="
$t = git diff --name-only origin/test origin/development -- src/extract_load/postgres/tables_ivv_verify.yml
if ($t) { "CHANGED: tables_ivv_verify.yml" } else { "unchanged" }
"===== 6. development into test merges cleanly? ====="
git merge-tree --write-tree --name-only origin/test origin/development 2>&1 | Out-Null
if ($LASTEXITCODE -eq 0) { "CLEAN" } else { "CONFLICTS" }
}
