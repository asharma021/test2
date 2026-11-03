& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
"===== Latest commits on development ====="
git log -6 --format='%h %ad %an : %s' --date=short origin/development
"===== Is the compile fix still on development? ====="
if (git show origin/development:dbt/tests/generic/no_deprecated_abbreviations.sql | Select-String -SimpleMatch "flags.WHICH == 'compile'" -Quiet) { Write-Host "YES - fix is on development" -ForegroundColor Green } else { Write-Host "NO - fix is missing on development" -ForegroundColor Red }
"===== Files changed by the latest merge into development ====="
git diff --stat origin/development~1 origin/development
"===== Last commits touching the test file ====="
git log -3 --format='%h %ad %an : %s' --date=short origin/development -- dbt/tests/generic/no_deprecated_abbreviations.sql
}
