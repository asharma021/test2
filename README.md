& {
# ===== DI-4659: E&L health check - v2 (ruff/pre-commit compliant) =====
$ErrorActionPreference = 'Continue'; $env:GIT_TERMINAL_PROMPT = '0'
$Repo  = 'C:\Users\SharmaAashishR\oa-business-layer'
$Br    = 'feature/el-failure-notifications'
$CiRel = '.gitlab-ci.yml'
$PyRel = 'scripts/el_health_check.py'
if (-not (Test-Path (Join-Path $Repo '.git'))) { Write-Host 'ABORT: not a git repo' -ForegroundColor Red; return }
Set-Location $Repo
function G { @(& git --no-pager -c core.quotepath=off @args 2>&1 | ForEach-Object { "$_" }) }
$CiPath = Join-Path $Repo $CiRel
$PyPath = Join-Path $Repo ($PyRel -replace '/', '\')
function PyTracked { [bool]((G ls-files -- $PyRel) -join '') }
function Revert([string]$why) {
  Write-Host "REVERTING: $why" -ForegroundColor Red
  G reset -q HEAD -- $CiRel $PyRel | Out-Null
  G checkout -- $CiRel | Out-Null
  if ((Test-Path $PyPath) -and -not (PyTracked)) { Remove-Item $PyPath -Force }
  Write-Host 'Reverted. git status:' -ForegroundColor Yellow; G status --short | Write-Host
}

# ---------- 0. clean up leftovers from the previous attempt ----------
G reset -q HEAD -- $CiRel $PyRel | Out-Null
G checkout -- $CiRel | Out-Null
if ((Test-Path $PyPath) -and -not (PyTracked)) { Remove-Item $PyPath -Force; Write-Host 'Removed leftover untracked scripts/el_health_check.py' -ForegroundColor Yellow }

# ---------- pre-checks ----------
G fetch origin $Br | Out-Null
if (((G rev-parse --abbrev-ref HEAD) -join '') -ne $Br) { Write-Host "ABORT: not on $Br" -ForegroundColor Red; return }
if (((G rev-parse HEAD) -join '') -ne ((G rev-parse "origin/$Br") -join '')) { Write-Host 'ABORT: HEAD != origin' -ForegroundColor Red; return }
$tracked = G status --porcelain | Where-Object { $_ -and $_ -notmatch '^\?\? ' }
if ($tracked) { Write-Host 'ABORT: uncommitted tracked changes:' -ForegroundColor Red; $tracked | Write-Host; return }
if (PyTracked) { Write-Host "ABORT: $PyRel already tracked" -ForegroundColor Red; return }
if (Select-String -Path $CiPath -Pattern '^el_health_check:' -Quiet) { Write-Host 'ABORT: el_health_check already in CI file' -ForegroundColor Red; return }
if (-not (Test-Path '.venv\Scripts\python.exe')) { Write-Host 'ABORT: .venv python not found' -ForegroundColor Red; return }
$py   = (Resolve-Path '.venv\Scripts\python.exe').Path
$ruff = if (Test-Path '.venv\Scripts\ruff.exe') { (Resolve-Path '.venv\Scripts\ruff.exe').Path } else { $null }
$pc   = if (Test-Path '.venv\Scripts\pre-commit.exe') { (Resolve-Path '.venv\Scripts\pre-commit.exe').Path } else { $null }
Write-Host "Clean start on $Br @ $((G rev-parse --short=8 HEAD) -join '') | ruff: $([bool]$ruff) | pre-commit: $([bool]$pc)" -ForegroundColor Cyan

# ---------- 1. write scripts/el_health_check.py ----------
$pyCode = @'
"""E&L health check for scheduled pipelines (DI-4659, DI-4663).

Reads logs/<job>.log and logs/<job>.status (the status file is written by the
.el_job after_script), writes el_health_summary.md, and exits 1 when an E&L job
or table failed and the job is not listed in EL_EXPECTED_FAILURES.
"""

from __future__ import annotations

import os
import re
import sys
from dataclasses import dataclass, field
from datetime import datetime
from pathlib import Path

JOBS = (
    "oracle_extract_load",
    "sqlserver_extract_load",
    "postgres_extract_load",
    "informix_extract_load",
    "prod_mirror",
)
NO_FAILURES = {"", "none", "-", "n/a", "[]"}
ERROR_RE = re.compile(r"\bERROR\b")


@dataclass
class JobResult:
    job: str
    status: str = "-"
    processed: str = "-"
    rows: str = "-"
    failed_tables: list[str] = field(default_factory=list)
    errors: list[str] = field(default_factory=list)
    result: str = "NOT RUN"


def _read(path: Path) -> str | None:
    if not path.is_file():
        return None
    return path.read_text(encoding="utf-8", errors="replace")


def _last(pattern: str, text: str) -> str | None:
    found = re.findall(pattern, text, flags=re.MULTILINE)
    return found[-1].strip() if found else None


def check_job(job: str, log_dir: Path) -> JobResult:
    res = JobResult(job=job)
    raw_status = _read(log_dir / f"{job}.status")
    log = _read(log_dir / f"{job}.log")
    if raw_status is None and log is None:
        return res
    status = raw_status.strip().lower() if raw_status else None
    res.status = status or "-"
    text = log or ""
    res.processed = _last(r"Tables processed:\s*([\d,]+)", text) or "-"
    res.rows = _last(r"Total rows loaded:\s*([\d,]+)", text) or "-"
    failed = _last(r"Failed tables:[ \t]*(.*)$", text)
    if failed is not None and failed.lower() not in NO_FAILURES:
        res.failed_tables = [t for t in re.split(r"[,\s]+", failed) if t]
    with_errors = _last(r"Tables with errors:\s*(\d+)", text)
    error_lines = [ln.strip() for ln in text.splitlines() if ERROR_RE.search(ln)]
    res.errors = error_lines[-5:]
    if log is None:
        res.errors.append("log file missing")
    problem = (
        status in {"failed", "canceled"}
        or bool(res.failed_tables)
        or (with_errors is not None and with_errors != "0")
    )
    res.result = "FAILED" if problem else "OK"
    return res


def build_summary(results: list[JobResult], problems: list[JobResult]) -> str:
    verdict = "FAILED" if problems else "PASSED"
    lines = [
        f"# E&L health check: {verdict}",
        "",
        f"- Pipeline: {os.environ.get('CI_PIPELINE_URL', 'local run')}",
        f"- Branch: {os.environ.get('CI_COMMIT_REF_NAME', '-')}",
        f"- Checked: {datetime.now().strftime('%Y-%m-%d %H:%M')}",
        "",
        "| Job | Result | Job status | Tables | Rows | Failed tables |",
        "|---|---|---|---|---|---|",
    ]
    for r in results:
        failed = ", ".join(r.failed_tables) or "-"
        cells = [r.job, r.result, r.status, r.processed, r.rows, failed]
        lines.append("| " + " | ".join(cells) + " |")
    for r in problems:
        details = r.errors or ["no ERROR lines found in log"]
        lines += ["", f"## {r.job}", *(f"- {e}" for e in details)]
    return "\n".join(lines) + "\n"


def main() -> int:
    reconfigure = getattr(sys.stdout, "reconfigure", None)
    if callable(reconfigure):
        reconfigure(errors="replace")
    log_dir = Path(os.environ.get("EL_LOG_DIR", "logs"))
    expected_raw = os.environ.get("EL_EXPECTED_FAILURES", "")
    expected = set(expected_raw.replace(",", " ").split())
    results = [check_job(job, log_dir) for job in JOBS]
    problems: list[JobResult] = []
    for r in results:
        if r.result == "FAILED":
            if r.job in expected:
                r.result = "EXPECTED FAIL"
            else:
                problems.append(r)
    summary = build_summary(results, problems)
    summary_file = Path(os.environ.get("EL_SUMMARY_FILE", "el_health_summary.md"))
    summary_file.write_text(summary, encoding="utf-8")
    print(summary)
    if problems:
        print("E&L health check FAILED: " + ", ".join(r.job for r in problems))
        return 1
    print("E&L health check passed.")
    return 0


if __name__ == "__main__":
    sys.exit(main())
'@
New-Item -ItemType Directory -Force -Path (Split-Path $PyPath) | Out-Null
[IO.File]::WriteAllText($PyPath, (($pyCode -replace "`r`n", "`n") + "`n"), (New-Object Text.UTF8Encoding($false)))

# ---------- 2. ruff format + lint (repo config) ----------
if ($ruff) {
  & $ruff format $PyRel 2>&1 | ForEach-Object { Write-Host "  ruff format: $_" }
  $rc = & $ruff check $PyRel 2>&1 | ForEach-Object { "$_" }
  if ($LASTEXITCODE -ne 0) { $rc | Write-Host; Revert 'ruff check failed'; return }
  Write-Host '  ruff check: passed' -ForegroundColor Green
}

# ---------- 3. edit .gitlab-ci.yml ----------
$orig = Join-Path $env:TEMP 'gitlab-ci.before.yml'
Copy-Item $CiPath $orig -Force
$bytes = [IO.File]::ReadAllBytes($CiPath)
$bom = ($bytes.Length -ge 3 -and $bytes[0] -eq 0xEF -and $bytes[1] -eq 0xBB -and $bytes[2] -eq 0xBF)
$text = [IO.File]::ReadAllText($CiPath)
$nl = if ($text.Contains("`r`n")) { "`r`n" } else { "`n" }
$L = New-Object System.Collections.Generic.List[string]
$L.AddRange([string[]]($text -split "\r?\n"))
$start = @(for ($i = 0; $i -lt $L.Count; $i++) { if ($L[$i] -match '^\.el_job:\s*(#.*)?$') { $i } })
if ($start.Count -ne 1) { Revert ".el_job header matches: $($start.Count)"; return }
$s = $start[0]; $e = $L.Count
for ($i = $s + 1; $i -lt $L.Count; $i++) { if ($L[$i] -match '^[^\s#]') { $e = $i; break } }
$logIdx   = @(for ($i = $s + 1; $i -lt $e; $i++) { if ($L[$i] -match '^\s+-\s+[''"]?logs/\$CI_JOB_NAME\.log[''"]?\s*$') { $i } })
$artIdx   = @(for ($i = $s + 1; $i -lt $e; $i++) { if ($L[$i] -match '^\s+artifacts:\s*$') { $i } })
$hasAfter = @(for ($i = $s + 1; $i -lt $e; $i++) { if ($L[$i] -match '^\s+after_script:') { $i } })
if ($logIdx.Count -ne 1 -or $artIdx.Count -ne 1 -or $hasAfter.Count -ne 0 -or $artIdx[0] -gt $logIdx[0]) {
  Revert ".el_job layout unexpected (log=$($logIdx.Count) artifacts=$($artIdx.Count) after_script=$($hasAfter.Count))"; return
}
$ind = ([regex]::Match($L[$artIdx[0]], '^(\s+)')).Groups[1].Value
$i2 = "$ind$ind"; $i3 = "$i2  "
$L.Insert($logIdx[0] + 1, ($L[$logIdx[0]] -replace '\.log', '.status'))
$cmd = '$s = if ($env:CI_JOB_STATUS) { $env:CI_JOB_STATUS } else { "unknown" }; New-Item -ItemType Directory -Force -Path logs | Out-Null; Set-Content -Path "logs/$($env:CI_JOB_NAME).status" -Value $s -Encoding ASCII'
$after = @(
  ($ind + '# Record the final job status for el_health_check (DI-4659).'),
  ($ind + 'after_script:'),
  ($ind + '  - ''' + $cmd + '''')
)
$L.InsertRange($artIdx[0], [string[]]$after)
$JobsList = 'oracle_extract_load', 'sqlserver_extract_load', 'postgres_extract_load', 'informix_extract_load', 'prod_mirror'
$hb = New-Object System.Collections.Generic.List[string]
$hb.Add('')
$hb.Add('# E&L health check (DI-4659 / DI-4663): reads each E&L job log + status file, writes')
$hb.Add('# el_health_summary.md, and fails the pipeline when an E&L job or table failed unexpectedly,')
$hb.Add('# so failures are not hidden by allow_failure. Known failures go in EL_EXPECTED_FAILURES.')
$hb.Add('el_health_check:')
$hb.Add($ind + 'stage: .post')
$hb.Add($ind + 'extends: .cache_venv')
$hb.Add($ind + 'timeout: 10 minutes')
$hb.Add($ind + 'needs:')
foreach ($j in $JobsList) { $hb.Add($i2 + '- job: ' + $j); $hb.Add($i3 + 'optional: true'); $hb.Add($i3 + 'artifacts: true') }
$hb.Add($ind + 'rules:')
$hb.Add($i2 + '- if: ''$CI_PIPELINE_SOURCE == "schedule" || $CI_PIPELINE_SOURCE == "web"''')
$hb.Add($i3 + 'when: always')
$hb.Add($i2 + '- when: never')
$hb.Add($ind + 'variables:')
$hb.Add($i2 + 'EL_EXPECTED_FAILURES: "informix_extract_load"')
$hb.Add($ind + 'script:')
$hb.Add($i2 + '- ''if (Get-Command uv -ErrorAction SilentlyContinue) { uv run --no-sync python scripts/el_health_check.py } else { python scripts/el_health_check.py }; exit $LASTEXITCODE''')
$hb.Add($ind + 'artifacts:')
$hb.Add($i2 + 'when: always')
$hb.Add($i2 + 'paths:')
$hb.Add($i3 + '- el_health_summary.md')
$hb.Add($i2 + 'expire_in: 30 days')
if ($L[$L.Count - 1] -eq '') { $L.InsertRange($L.Count - 1, [string[]]$hb) } else { $L.AddRange([string[]]$hb) }
[IO.File]::WriteAllText($CiPath, ($L -join $nl), (New-Object Text.UTF8Encoding($bom)))
$added = 1 + $after.Count + $hb.Count

# ---------- 4. repo pre-commit hooks on both files (run twice: fixers, then verify) ----------
if ($pc) {
  Write-Host ''; Write-Host '--- pre-commit ---' -ForegroundColor Cyan
  & $pc run --files $CiRel $PyRel 2>&1 | Out-Null
  $o = & $pc run --files $CiRel $PyRel 2>&1 | ForEach-Object { "$_" }
  if ($LASTEXITCODE -ne 0) { $o | Select-Object -Last 40 | Write-Host; Revert 'pre-commit hooks failed'; return }
  Write-Host '  pre-commit: all hooks passed' -ForegroundColor Green
}

# ---------- 5. self-test the final (formatted) script ----------
$testRoot = Join-Path $env:TEMP ('elhc_' + [guid]::NewGuid().ToString('N').Substring(0, 8))
function Run-Scenario([string]$name, [hashtable]$files, [int]$expectRc, [string[]]$mustMatch) {
  $d = Join-Path $testRoot $name; $logs = Join-Path $d 'logs'; $sumPath = Join-Path $d 'summary.md'
  New-Item -ItemType Directory -Force -Path $logs | Out-Null
  foreach ($k in $files.Keys) { [IO.File]::WriteAllText((Join-Path $logs $k), $files[$k]) }
  $env:EL_LOG_DIR = $logs; $env:EL_SUMMARY_FILE = $sumPath; $env:EL_EXPECTED_FAILURES = 'informix_extract_load'
  $out = & $py -I $PyPath 2>&1 | ForEach-Object { "$_" }; $rc = $LASTEXITCODE
  Remove-Item Env:EL_LOG_DIR, Env:EL_SUMMARY_FILE, Env:EL_EXPECTED_FAILURES -ErrorAction SilentlyContinue
  $sum = if (Test-Path $sumPath) { [IO.File]::ReadAllText($sumPath) } else { '' }
  $ok = ($rc -eq $expectRc); foreach ($m in $mustMatch) { if ($sum -notmatch $m) { $ok = $false } }
  Write-Host ("  {0,-14} rc={1} expected={2} -> {3}" -f $name, $rc, $expectRc, $(if ($ok) { 'PASS' } else { 'FAIL' })) -ForegroundColor $(if ($ok) { 'Green' } else { 'Red' })
  if (-not $ok) { $out | Write-Host }
  return $ok
}
$oraFail = "06:18:32 INFO Tables processed: 164`n06:18:32 INFO Total rows loaded: 145,761,327`n06:18:32 INFO Tables with errors: 1`n06:18:32 INFO Failed tables: vw_fact_sot_fst_returns`n06:18:32 ERROR Failed tables: vw_fact_sot_fst_returns`n"
$oraOk   = "06:18:32 INFO Tables processed: 163`n06:18:32 INFO Total rows loaded: 145,000,000`n06:18:32 INFO Tables with errors: 0`n"
Write-Host ''; Write-Host '--- health check self-test ---' -ForegroundColor Cyan
$t1 = Run-Scenario 'oracle_fail' @{ 'oracle_extract_load.status' = 'failed'; 'oracle_extract_load.log' = $oraFail; 'sqlserver_extract_load.status' = 'success'; 'sqlserver_extract_load.log' = "INFO done`n"; 'informix_extract_load.status' = 'failed'; 'informix_extract_load.log' = "ERROR not configured`n" } 1 @('oracle_extract_load \| FAILED', 'informix_extract_load \| EXPECTED FAIL', 'vw_fact_sot_fst_returns')
$t2 = Run-Scenario 'clean_run' @{ 'oracle_extract_load.status' = 'success'; 'oracle_extract_load.log' = $oraOk; 'sqlserver_extract_load.status' = 'success'; 'sqlserver_extract_load.log' = "INFO done`n"; 'informix_extract_load.status' = 'failed'; 'informix_extract_load.log' = "ERROR not configured`n" } 0 @('health check: PASSED', 'oracle_extract_load \| OK')
$t3 = Run-Scenario 'nothing_ran' @{} 0 @('oracle_extract_load \| NOT RUN', 'health check: PASSED')
$t4 = Run-Scenario 'table_fail_ok' @{ 'sqlserver_extract_load.status' = 'success'; 'sqlserver_extract_load.log' = "INFO Failed tables: dbo.some_table`n" } 1 @('sqlserver_extract_load \| FAILED')
Remove-Item $testRoot -Recurse -Force -ErrorAction SilentlyContinue
if (-not ($t1 -and $t2 -and $t3 -and $t4)) { Revert 'health check self-test failed'; return }

# ---------- 6. validate CI diff + YAML ----------
$num = (G diff --numstat -- $CiRel) -join ''
if ($num -ne "$added`t0`t$CiRel") { Revert "unexpected numstat: '$num' (expected $added added, 0 deleted)"; return }
$chk = Join-Path $env:TEMP 'check_ci_change.py'
@'
import sys, yaml
class L(yaml.SafeLoader): pass
def unk(loader, suffix, node):
    if isinstance(node, yaml.ScalarNode):   return "!%s %s" % (suffix, loader.construct_scalar(node))
    if isinstance(node, yaml.SequenceNode): return {"!" + suffix: loader.construct_sequence(node, deep=True)}
    return {"!" + suffix: loader.construct_mapping(node, deep=True)}
L.add_multi_constructor("!", unk)
def load(p):
    d = {}
    for x in yaml.load_all(open(p, encoding="utf-8-sig"), Loader=L):
        if isinstance(x, dict) and not ("spec" in x and len(x) == 1): d.update(x)
    return d
a, b = load(sys.argv[1]), load(sys.argv[2])
errs = []
if set(b) - set(a) != {"el_health_check"}: errs.append("added keys: %s" % (set(b) - set(a)))
if set(a) - set(b): errs.append("removed keys: %s" % (set(a) - set(b)))
for k in a:
    if k != ".el_job" and a[k] != b.get(k): errs.append("changed: " + k)
ea, eb = a[".el_job"], b[".el_job"]
changed = {k for k in set(ea) | set(eb) if ea.get(k) != eb.get(k)}
if changed != {"after_script", "artifacts"}: errs.append(".el_job changed keys: %s" % changed)
pa, pb = ea["artifacts"].get("paths", []), eb["artifacts"].get("paths", [])
if sorted(set(pb) - set(pa)) != ["logs/$CI_JOB_NAME.status"] or len(pb) != len(pa) + 1: errs.append("artifact paths: %s" % pb)
if {k: v for k, v in ea["artifacts"].items() if k != "paths"} != {k: v for k, v in eb["artifacts"].items() if k != "paths"}: errs.append("artifacts other keys changed")
asc = eb.get("after_script") or []
if len(asc) != 1 or "CI_JOB_STATUS" not in asc[0]: errs.append("after_script: %s" % asc)
h = b.get("el_health_check", {})
need = {n["job"] for n in h.get("needs", []) if isinstance(n, dict)}
if h.get("stage") != ".post": errs.append("stage: %s" % h.get("stage"))
if need != {"oracle_extract_load", "sqlserver_extract_load", "postgres_extract_load", "informix_extract_load", "prod_mirror"}: errs.append("needs: %s" % need)
for j in need:
    if j not in b: errs.append("needs unknown job: " + j)
if "allow_failure" in h: errs.append("allow_failure must not be set")
if not any("scripts/el_health_check.py" in s for s in h.get("script", [])): errs.append("script line")
print("YAML CHECK: " + ("OK" if not errs else "; ".join(errs)))
print("top-level keys before=%d after=%d" % (len(a), len(b)))
sys.exit(0 if not errs else 3)
'@ | Set-Content -Path $chk -Encoding ASCII
Write-Host ''; Write-Host '--- YAML validation ---' -ForegroundColor Cyan
& $py -I $chk $orig $CiPath 2>&1 | ForEach-Object { Write-Host "  $_" }
if ($LASTEXITCODE -ne 0) { Revert 'YAML validation failed'; return }

# ---------- 7. commit + push (ci.skip) ----------
G add -- $CiRel $PyRel | Out-Null
$staged = (G diff --cached --name-status | Sort-Object) -join '|'
if ($staged -ne "A`t$PyRel|M`t$CiRel") { Revert "unexpected staged files: $staged"; return }
$c = G commit -m 'Add E&L health check to surface E&L failures' -m 'Each E&L job now records its final status in logs/<job>.status (after_script in .el_job). New el_health_check job (.post, schedule/web pipelines) reads E&L logs and status files, writes el_health_summary.md, and fails the pipeline when an E&L job or table failed unexpectedly, so failures are no longer hidden by allow_failure and the standard failed-pipeline email is sent. Known failures are listed in EL_EXPECTED_FAILURES (currently informix_extract_load). Refs DI-4659, DI-4663.'
if ($LASTEXITCODE -ne 0) { $c | Select-Object -Last 40 | Write-Host; Revert 'commit failed'; return }
$p = G push -o ci.skip origin $Br
$p | Write-Host
if ($LASTEXITCODE -ne 0) { Write-Host 'PUSH FAILED - commit kept locally; send me a photo.' -ForegroundColor Red; return }

Write-Host ''
Write-Host '================ DONE ================' -ForegroundColor Green
Write-Host "Branch : $Br @ $((G rev-parse --short=8 HEAD) -join '')"
G log --oneline 'origin/development..HEAD' | ForEach-Object { Write-Host "  $_" }
Write-Host "CI lines added: $added (0 deleted) | new file: $PyRel"
}
