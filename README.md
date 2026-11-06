& {
    $ErrorActionPreference = 'Continue'
    Set-Location 'C:\Users\SharmaAashishR\oa-business-layer'
    $dirty = git status --porcelain | Where-Object { $_ -notmatch '\.gitlab-ci\.yml\.backup$' }
    if ($dirty) { Write-Host "STOP: uncommitted changes:`n$dirty" -ForegroundColor Red; return }
    git fetch origin
    if ($LASTEXITCODE -ne 0) { Write-Host "STOP: git fetch failed" -ForegroundColor Red; return }
    git checkout development
    if ($LASTEXITCODE -ne 0) { Write-Host "STOP: could not switch to development" -ForegroundColor Red; return }
    git merge --ff-only origin/development
    if ($LASTEXITCODE -ne 0) { Write-Host "STOP: local development has diverged from origin - not touching it" -ForegroundColor Red; return }

    $path = (Resolve-Path '.gitlab-ci.yml').Path
    $text = [System.IO.File]::ReadAllText($path)
    $nl = if ($text -match "`r`n") { "`r`n" } else { "`n" }
    $lines = [System.Collections.Generic.List[string]]::new([string[]]($text -split "`r?`n"))
    $totalBefore = @($lines | Where-Object { $_ -match '^\s+EL_ENV_FILE:' }).Count

    function Get-Block([string]$job) {
        $s = -1
        for ($i = 0; $i -lt $lines.Count; $i++) { if ($lines[$i] -match "^$([regex]::Escape($job)):\s*$") { $s = $i; break } }
        if ($s -lt 0) { return $null }
        $e = $lines.Count
        for ($j = $s + 1; $j -lt $lines.Count; $j++) { if ($lines[$j] -match '^\S') { $e = $j; break } }
        return @($s, $e)
    }

    $pg = Get-Block 'postgres_extract_load'
    if (-not $pg) { Write-Host "STOP: postgres_extract_load not found" -ForegroundColor Red; return }
    $pathLine = $null
    for ($i = $pg[0]; $i -lt $pg[1]; $i++) { if ($lines[$i] -match '^\s+(EL_ENV_FILE:.*)$') { $pathLine = $Matches[1].Trim(); break } }
    if (-not $pathLine) { Write-Host "STOP: no EL_ENV_FILE in postgres_extract_load" -ForegroundColor Red; return }

    $toRemove = @()
    foreach ($job in 'oracle_extract_load','sqlserver_extract_load','postgres_extract_load','informix_extract_load') {
        $b = Get-Block $job
        if (-not $b) { Write-Host "STOP: $job not found" -ForegroundColor Red; return }
        $hits = @()
        for ($i = $b[0]; $i -lt $b[1]; $i++) { if ($lines[$i] -match '^\s+EL_ENV_FILE:') { $hits += $i } }
        if ($hits.Count -ne 1) { Write-Host "STOP: $job has $($hits.Count) EL_ENV_FILE lines (expected 1)" -ForegroundColor Red; return }
        $toRemove += $hits
    }
    foreach ($i in ($toRemove | Sort-Object -Descending)) { $lines.RemoveAt($i) }

    $up = Get-Block '.el_upstream'
    if (-not $up) { Write-Host "STOP: .el_upstream not found" -ForegroundColor Red; git checkout -- .gitlab-ci.yml; return }
    $extIdx = -1; $hasVars = $false
    for ($i = $up[0]; $i -lt $up[1]; $i++) {
        if ($lines[$i] -match '^  extends:\s*\.el_job\s*$') { $extIdx = $i }
        if ($lines[$i] -match '^  variables:') { $hasVars = $true }
    }
    if ($extIdx -lt 0 -or $hasVars) { Write-Host "STOP: .el_upstream layout not as expected" -ForegroundColor Red; git checkout -- .gitlab-ci.yml; return }
    $lines.InsertRange($extIdx + 1, [string[]]@('  variables:', ('    ' + $pathLine)))

    [System.IO.File]::WriteAllText($path, ($lines -join $nl), (New-Object System.Text.UTF8Encoding($false)))

    # ---- Self-check ----
    $ok = $true
    $up = Get-Block '.el_upstream'
    $n = 0; for ($i = $up[0]; $i -lt $up[1]; $i++) { if ($lines[$i] -match '^\s+EL_ENV_FILE:') { $n++ } }
    if ($n -ne 1) { $ok = $false; Write-Host "CHECK FAILED: .el_upstream has $n EL_ENV_FILE lines" -ForegroundColor Red }
    foreach ($job in 'oracle_extract_load','sqlserver_extract_load','postgres_extract_load','informix_extract_load') {
        $b = Get-Block $job; $m = 0
        for ($i = $b[0]; $i -lt $b[1]; $i++) { if ($lines[$i] -match '^\s+EL_ENV_FILE:') { $m++ } }
        if ($m -ne 0) { $ok = $false; Write-Host "CHECK FAILED: $job still has EL_ENV_FILE" -ForegroundColor Red }
    }
    $totalAfter = @($lines | Where-Object { $_ -match '^\s+EL_ENV_FILE:' }).Count
    if ($totalAfter -ne ($totalBefore - 3)) { $ok = $false; Write-Host "CHECK FAILED: EL_ENV_FILE count $totalBefore -> $totalAfter (expected -3)" -ForegroundColor Red }
    $stat = git diff --numstat
    if ($stat -notmatch '^2\s+4\s+\.gitlab-ci\.yml$') { $ok = $false; Write-Host "CHECK FAILED: unexpected diff: $stat" -ForegroundColor Red }

    if (-not $ok) { Write-Host "Reverting - nothing committed" -ForegroundColor Yellow; git checkout -- .gitlab-ci.yml; return }

    git diff
    git add .gitlab-ci.yml
    git commit -m "Define EL_ENV_FILE once on .el_upstream" -m "Moves EL_ENV_FILE from the four upstream extract/load jobs (oracle, sqlserver, postgres, informix) to the shared .el_upstream template, so the env file path is set in one place. No change in behavior: every job extending .el_upstream still receives it."
    if ($LASTEXITCODE -ne 0) { Write-Host "STOP: commit failed - reverting" -ForegroundColor Red; git checkout -- .gitlab-ci.yml; return }

    git push origin development
    if ($LASTEXITCODE -eq 0) {
        Write-Host "`nDONE: pushed directly to development ($(git rev-parse --short HEAD))" -ForegroundColor Green
        return
    }

    # Direct push rejected: move the commit to a branch for an MR, restore local development
    Write-Host "`nDirect push to development rejected - moving commit to a branch for an MR" -ForegroundColor Yellow
    $branch = 'el-env-file-on-el-upstream'
    git branch -f $branch HEAD
    git reset --hard origin/development
    git push -u origin $branch
    if ($LASTEXITCODE -ne 0) { Write-Host "STOP: branch push failed too - commit saved locally on $branch" -ForegroundColor Red; return }
    Write-Host "DONE: open an MR from $branch into development" -ForegroundColor Green
}
