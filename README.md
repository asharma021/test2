& {
# ===== READ-ONLY: locate DABL.VW_FACT_SOT_FST_RETURNS in E&L config =====
$ErrorActionPreference = 'Continue'; $env:GIT_TERMINAL_PROMPT = '0'
$Repo = 'C:\Users\SharmaAashishR\oa-business-layer'
$View = 'vw_fact_sot_fst_returns'
$Branches = 'production','test','development','pipeline-testing'
if (-not (Test-Path (Join-Path $Repo '.git'))) { Write-Host 'ABORT: not a git repo' -ForegroundColor Red; return }
Set-Location $Repo
function G { @(& git --no-pager -c core.quotepath=off @args 2>&1 | ForEach-Object { "$_" }) }

$statusBefore = G status --porcelain --untracked-files=all
$headBefore   = (G rev-parse HEAD) -join ''
G fetch origin @Branches | Out-Null      # updates origin/* refs only

Write-Host "Current branch: $((G rev-parse --abbrev-ref HEAD) -join '') @ $((G rev-parse --short=8 HEAD) -join '')" -ForegroundColor Cyan
foreach ($b in $Branches) {
  $ref = "origin/$b"
  $sha = (G rev-parse --verify -q --short=8 $ref) -join ''
  Write-Host ''
  Write-Host "===== $ref @ $sha =====" -ForegroundColor Cyan
  if (-not $sha) { Write-Host '  (branch not found)'; continue }
  $hits = G grep -n -i -I $View $ref
  if (-not $hits) { Write-Host '  no references'; continue }
  $files = $hits | ForEach-Object { ($_ -replace "^$([regex]::Escape($ref)):",'') -replace ':\d+:.*$','' } | Sort-Object -Unique
  foreach ($f in $files) {
    $blob = (G rev-parse --short=8 "${ref}:$f") -join ''
    Write-Host "  FILE: $f  (blob $blob)" -ForegroundColor Yellow
    G grep -n -i -I -C 4 $View $ref -- $f | ForEach-Object { '    ' + ($_ -replace "^$([regex]::Escape($ref)):[^:-]+[:-]",'') } | Write-Host
  }
}

$same = (($statusBefore -join "`n") -eq ((G status --porcelain --untracked-files=all) -join "`n")) -and ($headBefore -eq ((G rev-parse HEAD) -join ''))
Write-Host ''
Write-Host "Working tree + HEAD unchanged: $same" -ForegroundColor Green
}
