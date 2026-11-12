& {
# ===== New branch from development + cherry-pick Oracle fix (DI-4653) - v2 =====
$ErrorActionPreference = 'Continue'; $env:GIT_TERMINAL_PROMPT = '0'
$Repo   = 'C:\Users\SharmaAashishR\oa-business-layer'
$File   = 'src/extract_load/oracle/tables.yml'
$Base   = 'origin/development'
$Br     = 'feature/el-failure-notifications'
$FixBr  = 'fix/oracle-el-disable-vw-fact-sot-fst-returns'
$Pat    = 'Disable VW_FACT_SOT_FST_RETURNS'
if (-not (Test-Path (Join-Path $Repo '.git'))) { Write-Host 'ABORT: not a git repo' -ForegroundColor Red; return }
Set-Location $Repo
function G { @(& git --no-pager -c core.quotepath=off @args 2>&1 | ForEach-Object { "$_" }) }
$created = $false; $startBranch = (G rev-parse --abbrev-ref HEAD) -join ''
function Revert([string]$why) {
  Write-Host "REVERTING: $why" -ForegroundColor Red
  G cherry-pick --abort | Out-Null
  G checkout -- . | Out-Null
  G switch $startBranch | Out-Null
  if ($created) { G branch -D $Br | Out-Null }
  Write-Host "Reverted. Now on: $((G rev-parse --abbrev-ref HEAD) -join '')" -ForegroundColor Yellow
}
function IsGoodFix([string]$c) {
  if (-not $c) { return $false }
  $parents = ((G rev-list --parents -n 1 $c) -join '') -split ' '
  $files   = G show --name-only --format= $c | Where-Object { $_ }
  $subj    = (G log -1 --format=%s $c) -join ''
  return ($parents.Count -eq 2) -and (@($files).Count -eq 1) -and ($files -eq $File) -and ($subj -like "*$Pat*")
}

# --- pre-checks ---
G fetch origin development production | Out-Null
$tracked = G status --porcelain | Where-Object { $_ -and $_ -notmatch '^\?\? ' }
if ($tracked) { Write-Host 'ABORT: uncommitted tracked changes:' -ForegroundColor Red; $tracked | Write-Host; return }
if ((G rev-parse --verify -q "refs/heads/$Br")) { Write-Host "ABORT: local branch $Br exists" -ForegroundColor Red; return }
if ((G ls-remote --heads origin $Br)) { Write-Host "ABORT: remote branch $Br exists" -ForegroundColor Red; return }

$rx = '^\s*- source_table: VW_FACT_SOT_FST_RETURNS\s*$'
$devHas  = -not ((G show "${Base}:$File") | Where-Object { $_ -match $rx })
$prodHas = -not ((G show "origin/production:$File") | Where-Object { $_ -match $rx })

# --- locate the real (non-merge) fix commit ---
$Fix = ''
foreach ($ref in @("refs/heads/$FixBr", "refs/remotes/origin/$FixBr")) {
  $c = (G rev-parse --verify -q --short=8 $ref) -join ''
  if (IsGoodFix $c) { $Fix = $c; $src = $ref; break }
}
if (-not $Fix) {
  $c = (G log origin/production --no-merges --format=%h -1 --grep=$Pat) -join ''
  if (IsGoodFix $c) { $Fix = $c; $src = 'origin/production (no-merges search)' }
}
Write-Host "Fix commit          : $Fix  [$src]"
Write-Host "Fix on production   : $prodHas"
Write-Host "Fix on development  : $devHas"
if (-not $prodHas) { Write-Host 'ABORT: production does not contain the fix.' -ForegroundColor Red; return }
if (-not $devHas -and -not $Fix) { Write-Host 'ABORT: no valid non-merge fix commit found.' -ForegroundColor Red; return }

# --- branch ---
$baseSha = (G rev-parse --short=8 $Base) -join ''
$sw = G switch -c $Br $Base
if ($LASTEXITCODE -ne 0) { Write-Host "ABORT: $sw" -ForegroundColor Red; return }
$created = $true

# --- cherry-pick ---
if ($devHas) {
  Write-Host 'Development already has the fix - no cherry-pick needed.' -ForegroundColor Yellow
} else {
  $cp = G cherry-pick -x $Fix
  if ($LASTEXITCODE -ne 0) { $cp | Write-Host; Revert 'cherry-pick failed/conflict'; return }
  $files = G diff --name-only "$Base..HEAD"
  $num   = (G diff --numstat "$Base..HEAD" -- $File) -join ''
  if ((@($files).Count -ne 1) -or ($files -ne $File) -or ($num -ne "4`t2`t$File")) { Revert "unexpected diff: files=$files numstat=$num"; return }
  $same = ((G rev-parse "HEAD:$File") -join '') -eq ((G rev-parse "origin/production:$File") -join '')
  Write-Host ''; Write-Host '--- DIFF vs development ---' -ForegroundColor Cyan
  G diff "$Base..HEAD" -- $File | Write-Host
  Write-Host "tables.yml identical to production: $same" -ForegroundColor $(if ($same) { 'Green' } else { 'Yellow' })
}

# --- push (ci.skip) ---
$p = G push -o ci.skip -u origin $Br
$p | Write-Host
if ($LASTEXITCODE -ne 0) { Write-Host 'PUSH FAILED - branch kept locally; send me a photo.' -ForegroundColor Red; return }

Write-Host ''
Write-Host '================ DONE ================' -ForegroundColor Green
Write-Host "Branch : $Br @ $((G rev-parse --short=8 HEAD) -join '')"
Write-Host "Base   : $Base @ $baseSha"
G log --oneline "$Base..HEAD" | ForEach-Object { Write-Host "  $_" }
Write-Host 'No MR yet - notification work goes on this branch first.' -ForegroundColor Cyan
}
