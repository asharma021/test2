& {
    $ErrorActionPreference = 'Continue'
    Set-Location 'C:\Users\SharmaAashishR\oa-business-layer'
    $dirty = git status --porcelain | Where-Object { $_ -notmatch '\.gitlab-ci\.yml\.backup$' }
    if ($dirty) { Write-Host "STOP: uncommitted changes:`n$dirty" -ForegroundColor Red; return }
    git fetch origin
    if ($LASTEXITCODE -ne 0) { Write-Host "STOP: git fetch failed" -ForegroundColor Red; return }

    $hit = git log origin/development --no-merges --format="%h|%s" -n 50 | Where-Object { $_ -match 'TEMP: manual verify-only run of upstream extract/loads' }
    if (@($hit).Count -ne 1) { Write-Host "STOP: expected 1 TEMP commit on development, found $(@($hit).Count)" -ForegroundColor Red; $hit; return }
    $sha = ($hit -split '\|')[0]
    Write-Host "TEMP commit: $sha" -ForegroundColor Cyan

    $branch = 'remove-temp-dev-verify-rule'
    git checkout -B $branch origin/development
    if ($LASTEXITCODE -ne 0) { Write-Host "STOP: could not create branch" -ForegroundColor Red; return }
    git revert --no-edit $sha
    if ($LASTEXITCODE -ne 0) { Write-Host "STOP: revert failed - aborting" -ForegroundColor Red; git revert --abort; git checkout development; return }

    # Self-check: TEMP rule gone, EL_ENV_FILE lines still present in all 3 jobs
    $after = [System.IO.File]::ReadAllText((Resolve-Path '.gitlab-ci.yml').Path)
    $ok = $after -notmatch 'TEMP: dev verify-only check'
    foreach ($job in 'oracle_extract_load','sqlserver_extract_load','informix_extract_load') {
        if ($after -notmatch "(?ms)^$job\:.*?EL_ENV_FILE:") { $ok = $false; Write-Host "CHECK FAILED: $job lost EL_ENV_FILE" -ForegroundColor Red }
    }
    $stat = git diff --numstat HEAD~1 HEAD
    if ($stat -notmatch '^0\s+6\s+\.gitlab-ci\.yml$') { $ok = $false; Write-Host "CHECK FAILED: unexpected diff: $stat" -ForegroundColor Red }
    if (-not $ok) { Write-Host "Undoing revert - nothing pushed" -ForegroundColor Yellow; git reset --hard origin/development; git checkout development; return }

    git show --stat HEAD
    git push -u origin $branch
    Write-Host "`nDONE: open an MR from $branch into development" -ForegroundColor Green
}
