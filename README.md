$ErrorActionPreference = 'Continue'
Set-Location 'C:\Users\SharmaAashishR\oa-business-layer'
git fetch origin
if ($LASTEXITCODE -ne 0) { Write-Host "STOP: git fetch failed - fix sign-in first" -ForegroundColor Red; return }
if (git status --porcelain) { Write-Host "STOP: uncommitted changes - commit or stash first" -ForegroundColor Red; return }

$branch = 'el-env-file-upstream-jobs'
git checkout -B $branch origin/development
if ($LASTEXITCODE -ne 0) { Write-Host "STOP: could not create branch" -ForegroundColor Red; return }

$path = (Resolve-Path '.gitlab-ci.yml').Path
$text = [System.IO.File]::ReadAllText($path)
$nl = if ($text -match "`r`n") { "`r`n" } else { "`n" }
$lines = [System.Collections.Generic.List[string]]::new([string[]]($text -split "`r?`n"))

function Get-Block([string]$job) {
    $start = -1
    for ($i = 0; $i -lt $lines.Count; $i++) { if ($lines[$i] -match "^$job\:\s*$") { $start = $i; break } }
    if ($start -lt 0) { return $null }
    $end = $lines.Count
    for ($j = $start + 1; $j -lt $lines.Count; $j++) { if ($lines[$j] -match '^\S') { $end = $j; break } }
    return @($start, $end)
}

# Take the exact EL_ENV_FILE value from postgres_extract_load
$pg = Get-Block 'postgres_extract_load'
if (-not $pg) { Write-Host "STOP: postgres_extract_load not found" -ForegroundColor Red; git checkout development; return }
$pgLine = $null
for ($i = $pg[0]; $i -lt $pg[1]; $i++) { if ($lines[$i] -match '^\s+EL_ENV_FILE:') { $pgLine = $lines[$i].Trim(); break } }
if (-not $pgLine) { Write-Host "STOP: no EL_ENV_FILE in postgres_extract_load" -ForegroundColor Red; git checkout development; return }

$added = 0
foreach ($job in 'oracle_extract_load','sqlserver_extract_load','informix_extract_load') {
    $b = Get-Block $job
    if (-not $b) { Write-Host "STOP: $job not found" -ForegroundColor Red; git checkout -- .gitlab-ci.yml; git checkout development; return }
    $has = $false; $modIdx = -1
    for ($i = $b[0]; $i -lt $b[1]; $i++) {
        if ($lines[$i] -match '^\s+EL_ENV_FILE:') { $has = $true }
        if ($lines[$i] -match '^(\s+)EL_MODULE:') { $modIdx = $i; $indent = $Matches[1] }
    }
    if ($has) { Write-Host "$job already has EL_ENV_FILE - skipped"; continue }
    if ($modIdx -lt 0) { Write-Host "STOP: $job has no EL_MODULE line" -ForegroundColor Red; git checkout -- .gitlab-ci.yml; git checkout development; return }
    $lines.Insert($modIdx + 1, "$indent$pgLine")
    $added++
    Write-Host "Added EL_ENV_FILE to $job" -ForegroundColor Green
}

[System.IO.File]::WriteAllText($path, ($lines -join $nl), (New-Object System.Text.UTF8Encoding($false)))

# ---- Self-check ----
$ok = $true
foreach ($job in 'oracle_extract_load','sqlserver_extract_load','informix_extract_load') {
    $b = Get-Block $job
    $n = 0; for ($i = $b[0]; $i -lt $b[1]; $i++) { if ($lines[$i] -match '^\s+EL_ENV_FILE:') { $n++ } }
    if ($n -ne 1) { $ok = $false; Write-Host "CHECK FAILED: $job has $n EL_ENV_FILE lines" -ForegroundColor Red }
}
$stat = git diff --numstat
if ($stat -notmatch "^$added\s+0\s+\.gitlab-ci\.yml$") { $ok = $false; Write-Host "CHECK FAILED: unexpected diff: $stat" -ForegroundColor Red }

if (-not $ok -or $added -eq 0) {
    Write-Host "Reverting - nothing committed" -ForegroundColor Yellow
    git checkout -- .gitlab-ci.yml; git checkout development; git branch -D $branch
    return
}

git diff
git add .gitlab-ci.yml
git commit -m "Load EL_ENV_FILE in Oracle, SQL Server and Informix extract/load jobs" -m "Only postgres_extract_load set EL_ENV_FILE, so the other upstream jobs could not read their source connection settings and failed on missing INFORMIX_HOST / MSSQL_USER. Uses the same env file path as postgres_extract_load."
git push -u origin $branch
Write-Host "`nDONE: open an MR from $branch into development" -ForegroundColor Green
