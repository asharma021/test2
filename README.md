& {
# ===== Disable VW_FACT_SOT_FST_RETURNS in Oracle E&L config (DI-4653) =====
$ErrorActionPreference = 'Continue'; $env:GIT_TERMINAL_PROMPT = '0'
$Repo = 'C:\Users\SharmaAashishR\oa-business-layer'
$File = 'src/extract_load/oracle/tables.yml'
$Base = 'origin/development'
$Br   = 'fix/oracle-el-disable-vw-fact-sot-fst-returns'
if (-not (Test-Path (Join-Path $Repo '.git'))) { Write-Host 'ABORT: not a git repo' -ForegroundColor Red; return }
Set-Location $Repo
function G { @(& git --no-pager -c core.quotepath=off @args 2>&1 | ForEach-Object { "$_" }) }
$created = $false
function Revert([string]$why) {
  Write-Host "REVERTING: $why" -ForegroundColor Red
  G checkout -- $File | Out-Null
  G switch development | Out-Null
  if ($created) { G branch -D $Br | Out-Null }
  Write-Host "Reverted. Now on: $((G rev-parse --abbrev-ref HEAD) -join '')" -ForegroundColor Yellow
}

# --- pre-checks ---
G fetch origin development | Out-Null
$tracked = G status --porcelain | Where-Object { $_ -and $_ -notmatch '^\?\? ' }
if ($tracked) { Write-Host 'ABORT: uncommitted tracked changes:' -ForegroundColor Red; $tracked | Write-Host; return }
if ((G rev-parse --verify -q "refs/heads/$Br")) { Write-Host "ABORT: branch $Br already exists locally" -ForegroundColor Red; return }
if ((G ls-remote --heads origin $Br)) { Write-Host "ABORT: branch $Br already exists on origin" -ForegroundColor Red; return }
$baseSha = (G rev-parse --short=8 $Base) -join ''
Write-Host "Base $Base @ $baseSha" -ForegroundColor Cyan

# --- branch ---
$sw = G switch -c $Br $Base
if ($LASTEXITCODE -ne 0) { Write-Host "ABORT: could not create branch: $sw" -ForegroundColor Red; return }
$created = $true
if (((G rev-parse --short=8 HEAD) -join '') -ne $baseSha) { Revert 'HEAD != base'; return }

# --- edit (preserve BOM + line endings) ---
$path  = Join-Path $Repo $File
$bytes = [IO.File]::ReadAllBytes($path)
$bom   = ($bytes.Length -ge 3 -and $bytes[0] -eq 0xEF -and $bytes[1] -eq 0xBB -and $bytes[2] -eq 0xBF)
$orig  = Join-Path $env:TEMP 'tables_oracle_before.yml'
Copy-Item $path $orig -Force
$text  = [IO.File]::ReadAllText($path)
$nl    = if ($text.Contains("`r`n")) { "`r`n" } else { "`n" }
$rx    = [regex]'(?m)^( *)- source_table: VW_FACT_SOT_FST_RETURNS[ \t]*\r?\n\1  tags: \[tad_legacy, returns\][ \t]*\r?\n'
$ms    = $rx.Matches($text)
if ($ms.Count -ne 1) { Revert "expected exactly 1 match, found $($ms.Count)"; return }
$m  = $ms[0]; $in = $m.Groups[1].Value
$new = "$in# Disabled 2026-11-10 (DI-4653): BL view, not a source table; its Rcvd_Date logic fails with ORA-01841.$nl" +
       "$in# Re-enable only after the view SQL is fixed.$nl" +
       "$in# - source_table: VW_FACT_SOT_FST_RETURNS$nl" +
       "$in#   tags: [tad_legacy, returns]$nl"
$text = $text.Substring(0, $m.Index) + $new + $text.Substring($m.Index + $m.Length)
[IO.File]::WriteAllText($path, $text, (New-Object Text.UTF8Encoding($bom)))

# --- validate diff ---
$changed = G status --porcelain | Where-Object { $_ -and $_ -notmatch '^\?\? ' }
if ((@($changed).Count -ne 1) -or ($changed -notmatch [regex]::Escape($File))) { Revert "unexpected changed files: $changed"; return }
$num = (G diff --numstat -- $File) -join ''
if ($num -ne "4`t2`t$File") { Revert "unexpected numstat: $num"; return }
Write-Host ''; Write-Host '--- DIFF ---' -ForegroundColor Cyan
G diff -- $File | Write-Host

# --- YAML parse check ---
$py = $null
if (Test-Path '.venv\Scripts\python.exe') { $py = (Resolve-Path '.venv\Scripts\python.exe').Path }
elseif (Get-Command py -ErrorAction SilentlyContinue) { $py = 'py' }
elseif (Get-Command python -ErrorAction SilentlyContinue) { $py = 'python' }
$chk = Join-Path $env:TEMP 'check_tables_yml.py'
@'
import sys
try:
    import yaml
except ImportError:
    print("SKIP: pyyaml not installed"); sys.exit(4)
def walk(o, acc):
    if isinstance(o, dict):
        if "source_table" in o: acc.append(str(o["source_table"]).upper())
        for v in o.values(): walk(v, acc)
    elif isinstance(o, list):
        for v in o: walk(v, acc)
old, new = [], []
walk(yaml.safe_load(open(sys.argv[1], encoding="utf-8-sig")), old)
walk(yaml.safe_load(open(sys.argv[2], encoding="utf-8-sig")), new)
removed = sorted(set(old) - set(new)); added = sorted(set(new) - set(old))
print(f"YAML OK: before={len(old)} after={len(new)} removed={removed} added={added}")
ok = len(new) == len(old) - 1 and removed == ["VW_FACT_SOT_FST_RETURNS"] and not added
sys.exit(0 if ok else 3)
'@ | Set-Content -Path $chk -Encoding ASCII
if ($py) {
  $out = & $py -I $chk $orig $path 2>&1 | ForEach-Object { "$_" }
  $rc = $LASTEXITCODE; $out | Write-Host
  if ($rc -eq 4) { Write-Host 'WARN: YAML parse check skipped (no pyyaml); text checks passed.' -ForegroundColor Yellow }
  elseif ($rc -ne 0) { Revert "YAML check failed (rc=$rc)"; return }
} else { Write-Host 'WARN: no Python found; YAML parse check skipped; text checks passed.' -ForegroundColor Yellow }

# --- commit + push (ci.skip) ---
G add -- $File | Out-Null
$c = G commit -m 'Disable VW_FACT_SOT_FST_RETURNS in Oracle E&L config' -m 'DABL.VW_FACT_SOT_FST_RETURNS is a BL view, not a source table. Its Rcvd_Date logic fails with ORA-01841 on a full read, which fails the daily Oracle E&L. Commented out until the view SQL is fixed (DI-4653).'
if ($LASTEXITCODE -ne 0) { Write-Host $c; Revert 'commit failed'; return }
$p = G push -o ci.skip -u origin $Br
$p | Write-Host
if ($LASTEXITCODE -ne 0) { Write-Host 'PUSH FAILED - commit kept locally on branch; send me a photo.' -ForegroundColor Red; return }

Write-Host ''
Write-Host '================ DONE ================' -ForegroundColor Green
Write-Host "Branch : $Br @ $((G rev-parse --short=8 HEAD) -join '')"
Write-Host "Base   : $Base @ $baseSha"
Write-Host "Diff   : $num"
Write-Host 'Open MR (source -> development):' -ForegroundColor Cyan
Write-Host "https://gitlab.ad.ttb.gov/office-of-analytics/oa/oa-business-layer/-/merge_requests/new?merge_request[source_branch]=$Br&merge_request[target_branch]=development"
}
