$ErrorActionPreference = 'Continue'
Set-Location 'C:\Users\SharmaAashishR\oa-business-layer'
if (git status --porcelain) { Write-Host "STOP: uncommitted changes - commit or stash first" -ForegroundColor Red; return }
git checkout el-env-file-upstream-jobs
if ($LASTEXITCODE -ne 0) { Write-Host "STOP: could not switch branch" -ForegroundColor Red; return }
git pull --ff-only
if ($LASTEXITCODE -ne 0) { Write-Host "STOP: could not update branch" -ForegroundColor Red; return }

$path = (Resolve-Path '.gitlab-ci.yml').Path
$text = [System.IO.File]::ReadAllText($path)
if ($text -match 'TEMP: dev verify-only check') { Write-Host "Temp rule already present - nothing to do" -ForegroundColor Yellow; return }
$nl = if ($text -match "`r`n") { "`r`n" } else { "`n" }
$lines = [System.Collections.Generic.List[string]]::new([string[]]($text -split "`r?`n"))

# Locate .el_upstream block
$start = -1
for ($i = 0; $i -lt $lines.Count; $i++) { if ($lines[$i] -match '^\.el_upstream:\s*$') { $start = $i; break } }
if ($start -lt 0) { Write-Host "STOP: .el_upstream not found" -ForegroundColor Red; return }
$end = $lines.Count
for ($j = $start + 1; $j -lt $lines.Count; $j++) { if ($lines[$j] -match '^\S') { $end = $j; break } }

# Indent of the production rule, and last non-blank line of the block
$indent = $null
for ($i = $start; $i -lt $end; $i++) {
    if ($lines[$i] -match '^(\s*)- if: ''\$CI_COMMIT_BRANCH == \$CI_DEFAULT_BRANCH''') { $indent = $Matches[1] }
}
if ($null -eq $indent) { Write-Host "STOP: production rule not found in .el_upstream" -ForegroundColor Red; return }
$last = $end - 1
while ($last -gt $start -and [string]::IsNullOrWhiteSpace($lines[$last])) { $last-- }

$new = @(
    $indent + '# TEMP: dev verify-only check of upstream source settings. Remove before promoting to test.',
    $indent + '- if: ''$CI_COMMIT_BRANCH == "development" && $CI_PIPELINE_SOURCE == "web"''',
    $indent + '  variables:',
    $indent + '    DBT_TARGET: "dev"',
    $indent + '    EL_VERIFY_ONLY: "true"',
    $indent + '  when: manual'
)
$lines.InsertRange($last + 1, [string[]]$new)
[System.IO.File]::WriteAllText($path, ($lines -join $nl), (New-Object System.Text.UTF8Encoding($false)))

# ---- Self-check ----
$stat = git diff --numstat
$ok = ($stat -match '^6\s+0\s+\.gitlab-ci\.yml$')
$after = [System.IO.File]::ReadAllText($path)
if (($after -split 'TEMP: dev verify-only check').Count -ne 2) { $ok = $false }
if ($after -notmatch "(?m)^\s+- if: '\`$CI_COMMIT_BRANCH == \`$CI_DEFAULT_BRANCH'") { $ok = $false }

if (-not $ok) {
    Write-Host "CHECK FAILED ($stat) - reverting, nothing committed" -ForegroundColor Red
    git checkout -- .gitlab-ci.yml
    return
}

git diff
git add .gitlab-ci.yml
git commit -m "TEMP: manual verify-only run of upstream extract/loads on development" -m "Lets oracle, sqlserver and informix extract/load run manually on development in verify-only mode (no load) to confirm EL_ENV_FILE provides their source settings. To be reverted before promoting to test."
if ($LASTEXITCODE -ne 0) { Write-Host "STOP: commit failed (see pre-commit output) - reverting" -ForegroundColor Red; git checkout -- .gitlab-ci.yml; return }
git push
Write-Host "`nDONE. TEMP commit: $(git rev-parse --short HEAD)  <- save this, it is what we revert later" -ForegroundColor Green
