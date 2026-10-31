Set-Location C:\Users\SharmaAashishR\oa-business-layer

# ---- A. Stop anything still holding .venv open, then repair it ----
Get-Process | Where-Object { $_.Path -like "$PWD\.venv\*" } |
  Select-Object Id, ProcessName, Path | Format-Table -AutoSize
Get-Process | Where-Object { $_.Path -like "$PWD\.venv\*" } | Stop-Process -Force
uv sync
# If uv sync still fails, rebuild the local venv (local only, repo untouched):
#   Remove-Item .venv -Recurse -Force; uv sync
uv run --no-sync python -c "import yaml; print('pyyaml OK', yaml.__version__)"

# ---- B. Project path and default branch from git, not placeholders ----
$GitLab  = 'https://gitlab.ad.ttb.gov'
$remote  = git remote get-url origin
$ProjectPath = ($remote -replace '^(https?://[^/]+/|git@[^:]+:)', '') -replace '\.git$', ''
$ProjEnc = [uri]::EscapeDataString($ProjectPath)
git remote set-head origin --auto | Out-Null
$DefaultBranch = (git symbolic-ref --short refs/remotes/origin/HEAD) -replace '^origin/', ''
"Project path  : $ProjectPath"
"Default branch: $DefaultBranch"
if ([string]::IsNullOrWhiteSpace($env:GITLAB_TOKEN)) { Write-Warning 'GITLAB_TOKEN not set - step E will 404' }
$Hdr = @{ 'PRIVATE-TOKEN' = $env:GITLAB_TOKEN }
$Ci  = Join-Path $PWD '.gitlab-ci.yml'

# ---- C. Phase 1 structural check (script already in TEMP; --no-sync = no venv changes) ----
git diff -U0 -- .gitlab-ci.yml
uv run --no-sync python "$env:TEMP\check_p1.py" $Ci

# ---- D. Approved prod postgres_extract_load on the default branch ----
git fetch origin
$prodCi = git show "origin/${DefaultBranch}:.gitlab-ci.yml"
$s = ($prodCi | Select-String -Pattern '^postgres_extract_load:').LineNumber
if ($s) { $prodCi[($s-1)..($s+8)] } else { 'postgres_extract_load not found on default branch' }

# ---- E. CI Lint dry run - PostgreSQL jobs only ----
$content = Get-Content $Ci -Raw
foreach ($ref in @('pipeline-testing', $DefaultBranch)) {
  $body = @{ content = $content; dry_run = $true; include_jobs = $true; ref = $ref } | ConvertTo-Json -Depth 5
  $lint = Invoke-RestMethod -Method Post -Uri "$GitLab/api/v4/projects/$ProjEnc/ci/lint" -Headers $Hdr -ContentType 'application/json' -Body $body
  "`n=== Lint ref=$ref  valid=$($lint.valid) ==="
  $lint.errors; $lint.warnings
  $lint.jobs | Where-Object { $_.name -like '*postgres*' } |
    Select-Object name, stage, when, allow_failure | Format-Table -AutoSize
}
