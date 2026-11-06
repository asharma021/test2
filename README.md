& {
    $ErrorActionPreference = 'Continue'
    Set-Location 'C:\Users\SharmaAashishR\oa-business-layer'
    $stat = git diff --numstat
    if ($stat -notmatch '^6\s+0\s+\.gitlab-ci\.yml$') { Write-Host "STOP: unexpected diff: $stat" -ForegroundColor Red; return }

    git add .gitlab-ci.yml
    git commit -m "TEMP: manual verify-only run of upstream extract/loads on development" -m "Lets oracle, sqlserver and informix extract/load run manually on development in verify-only mode (no load) to confirm EL_ENV_FILE provides their source settings. To be reverted before promoting to test."
    if ($LASTEXITCODE -ne 0) { Write-Host "STOP: commit failed - see pre-commit output above. File left as is." -ForegroundColor Red; return }

    git push
    if ($LASTEXITCODE -ne 0) { Write-Host "STOP: push failed - commit is local only" -ForegroundColor Red; return }

    Write-Host "`nTEMP commit (revert this one later): $(git rev-parse --short HEAD)" -ForegroundColor Green
    Write-Host "EL_ENV_FILE fix commit (keep):     $(git rev-parse --short HEAD~1)" -ForegroundColor Green
    git log --oneline -3
}
