& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
if (git status --porcelain --untracked-files=no) { Write-Host "Uncommitted changes - nothing done." -ForegroundColor Red; return }
$start = (git rev-parse --abbrev-ref HEAD).Trim()
git switch --quiet feature/prod-el-readiness
if ((git rev-parse HEAD).Trim() -ne (git rev-parse origin/feature/prod-el-readiness).Trim()) { Write-Host "Local feature branch differs from origin - stopping." -ForegroundColor Red; git switch --quiet $start; return }
"===== MR !6 commit ====="
git show --stat --format='%h %s' HEAD
"===== dbt parse --target prod (no DB connection) ====="
Push-Location dbt
uv run --no-sync dbt parse --target prod --profiles-dir . --target-path target\prod_parse_check
$ok = ($LASTEXITCODE -eq 0)
Pop-Location
git switch --quiet $start
Write-Host "dbt parse --target prod on MR !6: $(if ($ok) { 'PASSED' } else { 'FAILED' })" -ForegroundColor $(if ($ok) { 'Green' } else { 'Red' })
"Back on: $(git rev-parse --abbrev-ref HEAD)"
}
