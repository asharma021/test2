& {
# ===== DI-4662: E&L env pre-flight check - v2 (ty-clean import) =====
$ErrorActionPreference = 'Continue'; $env:GIT_TERMINAL_PROMPT = '0'
$Repo  = 'C:\Users\SharmaAashishR\oa-business-layer'
$Br    = 'feature/el-failure-notifications'
$CiRel = '.gitlab-ci.yml'
$PyRel = 'scripts/el_env_check.py'
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

# ---------- pre-checks ----------
G fetch origin $Br | Out-Null
if (((G rev-parse --abbrev-ref HEAD) -join '') -ne $Br) { Write-Host "ABORT: not on $Br" -ForegroundColor Red; return }
if (((G rev-parse HEAD) -join '') -ne ((G rev-parse "origin/$Br") -join '')) { Write-Host 'ABORT: HEAD != origin' -ForegroundColor Red; return }
$tracked = G status --porcelain | Where-Object { $_ -and $_ -notmatch '^\?\? ' }
if ($tracked) { Write-Host 'ABORT: uncommitted tracked changes:' -ForegroundColor Red; $tracked | Write-Host; return }
if ((Test-Path $PyPath) -and -not (PyTracked)) { Remove-Item $PyPath -Force }
if (PyTracked) { Write-Host "ABORT: $PyRel already tracked" -ForegroundColor Red; return }
if (Select-String -Path $CiPath -Pattern 'el_env_check' -Quiet) { Write-Host 'ABORT: el_env_check already in CI file' -ForegroundColor Red; return }
if (-not (Test-Path '.venv\Scripts\python.exe')) { Write-Host 'ABORT: .venv python not found' -ForegroundColor Red; return }
$py   = (Resolve-Path '.venv\Scripts\python.exe').Path
$ruff = if (Test-Path '.venv\Scripts\ruff.exe') { (Resolve-Path '.venv\Scripts\ruff.exe').Path } else { $null }
$pc   = if (Test-Path '.venv\Scripts\pre-commit.exe') { (Resolve-Path '.venv\Scripts\pre-commit.exe').Path } else { $null }
Write-Host "Clean start on $Br @ $((G rev-parse --short=8 HEAD) -join '')" -ForegroundColor Cyan

# ---------- 1. write scripts/el_env_check.py ----------
$pyCode = @'
"""Pre-flight check of extract/load settings (DI-4662).

Runs at the start of every E&L job, before any data moves. Loads the EL env
file the same way the loaders do and fails fast, with a clear message, when the
file cannot be used or a setting the job needs is missing. Values are never
printed. Set EL_ENV_CHECK=off to bypass.
"""

from __future__ import annotations

import os
import sys
from pathlib import Path

from extract_load.common import env as el_env

# The loaders' own configuration error class(es), found by type so this check
# keeps working if the class is renamed.
CONFIG_ERRORS: tuple[type[Exception], ...] = tuple(
    obj for obj in vars(el_env).values() if isinstance(obj, type) and issubclass(obj, Exception)
) or (RuntimeError,)

# Settings each source needs that have no default in its connections module.
REQUIRED: dict[str, tuple[str, ...]] = {
    "oracle": ("ORACLE_USER", "ORACLE_PASSWORD"),
    "sqlserver": ("MSSQL_USER", "MSSQL_PASSWORD"),
    "postgres": ("SRC_PG_HOST", "SRC_PG_USER", "SRC_PG_PASSWORD"),
    "informix": ("INFORMIX_HOST", "INFORMIX_SERVER", "INFORMIX_USER", "INFORMIX_PASSWORD"),
    "mirror": ("MIRROR_PG_HOST", "MIRROR_PG_DATABASE", "MIRROR_PG_USER", "MIRROR_PG_PASSWORD"),
}
OFF_VALUES = {"off", "false", "0", "no"}


def source_for(job: str) -> str | None:
    override = os.environ.get("EL_CHECK_SOURCE", "").strip().lower()
    if override:
        return override
    if job.startswith("prod_mirror"):
        return "mirror"
    for source in ("oracle", "sqlserver", "postgres", "informix"):
        if job.startswith(f"{source}_"):
            return source
    return None


def _is_set(name: str) -> bool:
    value = (os.environ.get(name) or "").strip()
    return bool(value) and not (value.startswith("<") and value.endswith(">"))


def _profile_variants(name: str) -> list[str]:
    prefix = f"{name}_"
    return sorted(k for k in os.environ if k.startswith(prefix) and _is_set(k))


def run_checks(job: str) -> tuple[list[str], list[str]]:
    errors: list[str] = []
    warnings: list[str] = []
    try:
        path = el_env.load_el_env()
    except CONFIG_ERRORS as exc:
        return [f"EL env file not usable: {exc}"], warnings
    if path is None:
        warnings.append("no EL env file found; using the process environment only")
    for name in REQUIRED.get(source_for(job) or "", ()):
        if _is_set(name):
            continue
        variants = _profile_variants(name)
        if variants:
            found = ", ".join(variants)
            warnings.append(f"{name} not set; only profile variants found: {found}")
        else:
            errors.append(f"{name} is missing or empty in the EL env file")
    for check in (el_env.target_credentials, el_env.target_endpoint):
        try:
            check()
        except CONFIG_ERRORS as exc:
            errors.append(f"load target: {exc}")
    return errors, warnings


def main() -> int:
    reconfigure = getattr(sys.stdout, "reconfigure", None)
    if callable(reconfigure):
        reconfigure(errors="replace")
    job = os.environ.get("CI_JOB_NAME", "local")
    if os.environ.get("EL_ENV_CHECK", "").strip().lower() in OFF_VALUES:
        print(f"EL env check skipped for {job} (EL_ENV_CHECK=off)")
        return 0
    errors, warnings = run_checks(job)
    for w in warnings:
        print(f"WARNING EL env check ({job}): {w}")
    if not errors:
        print(f"EL env check passed for {job} (source: {source_for(job) or 'n/a'})")
        return 0
    lines = [f"ERROR EL env check ({job}): {e}" for e in errors]
    log_dir = Path(os.environ.get("EL_LOG_DIR", "logs"))
    log_dir.mkdir(parents=True, exist_ok=True)
    with (log_dir / f"{job}.log").open("a", encoding="utf-8") as fh:
        fh.write("\n".join(lines) + "\n")
    print("\n".join(lines))
    print(f"EL env check FAILED for {job}: fix the EL env file before rerunning.")
    return 1


if __name__ == "__main__":
    sys.exit(main())
'@
New-Item -ItemType Directory -Force -Path (Split-Path $PyPath) | Out-Null
[IO.File]::WriteAllText($PyPath, (($pyCode -replace "`r`n", "`n") + "`n"), (New-Object Text.UTF8Encoding($false)))

if ($ruff) {
  & $ruff format $PyRel 2>&1 | ForEach-Object { Write-Host "  ruff format: $_" }
  $rc = & $ruff check $PyRel 2>&1 | ForEach-Object { "$_" }
  if ($LASTEXITCODE -ne 0) { $rc | Write-Host; Revert 'ruff check failed'; return }
  Write-Host '  ruff check: passed' -ForegroundColor Green
}

# ---------- 2. edit .gitlab-ci.yml (.el_job: run check right before the loader) ----------
$orig = Join-Path $env:TEMP 'gitlab-ci.before-envcheck.yml'
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
$runIdx = @(for ($i = $s + 1; $i -lt $e; $i++) { if ($L[$i] -match '^\s+(-\s+)?uv run python @elArgs\s*$') { $i } })
if ($runIdx.Count -ne 1) { Revert "expected 1 'uv run python @elArgs' line in .el_job, found $($runIdx.Count)"; return }
$m = [regex]::Match($L[$runIdx[0]], '^(\s+)(-\s+)?')
$indent = $m.Groups[1].Value; $isItem = $m.Groups[2].Success
if ($isItem) {
  $new = @(
    ($indent + '# Pre-flight: fail fast if the EL env file or a required setting is missing (DI-4662).'),
    ($indent + '- uv run python scripts/el_env_check.py')
  )
} else {
  $new = @(
    ($indent + '# Pre-flight: fail fast if the EL env file or a required setting is missing (DI-4662).'),
    ($indent + 'uv run python scripts/el_env_check.py; if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }')
  )
}
$L.InsertRange($runIdx[0], [string[]]$new)
[IO.File]::WriteAllText($CiPath, ($L -join $nl), (New-Object Text.UTF8Encoding($bom)))

# ---------- 3. pre-commit on both files ----------
if ($pc) {
  Write-Host ''; Write-Host '--- pre-commit ---' -ForegroundColor Cyan
  & $pc run --files $CiRel $PyRel 2>&1 | Out-Null
  $o = & $pc run --files $CiRel $PyRel 2>&1 | ForEach-Object { "$_" }
  if ($LASTEXITCODE -ne 0) { $o | Select-Object -Last 40 | Write-Host; Revert 'pre-commit hooks failed'; return }
  Write-Host '  pre-commit: all hooks passed' -ForegroundColor Green
}

# ---------- 4. self-test (fake env files; real loader code; secrets must never print) ----------
$testRoot = Join-Path $env:TEMP ('elenv_' + [guid]::NewGuid().ToString('N').Substring(0, 8))
New-Item -ItemType Directory -Force -Path $testRoot | Out-Null
$touch = @('EL_ENV_FILE', 'CI_JOB_NAME', 'DBT_TARGET', 'EL_LOG_DIR', 'EL_ENV_CHECK', 'EL_CHECK_SOURCE',
  'ORACLE_USER', 'ORACLE_PASSWORD', 'MSSQL_USER', 'MSSQL_PASSWORD', 'MSSQL_USER_QUEST', 'MSSQL_PASSWORD_QUEST',
  'TARGET_PG_USER', 'TARGET_PG_PASSWORD')
$saved = @{}; foreach ($k in $touch) { $saved[$k] = [Environment]::GetEnvironmentVariable($k, 'Process') }
$secrets = @('SuperSecret123', 'TargetSecret456', 'QuestSecret789')
function Run-Scenario([string]$name, [string]$envContent, [hashtable]$vars, [int]$expectRc, [string[]]$mustMatch) {
  foreach ($k in $touch) { [Environment]::SetEnvironmentVariable($k, $null, 'Process') }
  $d = Join-Path $testRoot $name; New-Item -ItemType Directory -Force -Path $d | Out-Null
  $envPath = Join-Path $d 'extract_load.env'
  if ($envContent) { [IO.File]::WriteAllText($envPath, $envContent) }
  $env:EL_ENV_FILE = $envPath; $env:DBT_TARGET = 'dev'; $env:EL_LOG_DIR = (Join-Path $d 'logs')
  foreach ($k in $vars.Keys) { [Environment]::SetEnvironmentVariable($k, $vars[$k], 'Process') }
  $out = (& $py -I $PyPath 2>&1 | ForEach-Object { "$_" }) -join "`n"; $rc = $LASTEXITCODE
  $ok = ($rc -eq $expectRc)
  foreach ($mm in $mustMatch) { if ($out -notmatch $mm) { $ok = $false } }
  foreach ($sec in $secrets) { if ($out -match $sec) { $ok = $false; Write-Host "  SECRET LEAKED in $name" -ForegroundColor Red } }
  Write-Host ("  {0,-14} rc={1} expected={2} -> {3}" -f $name, $rc, $expectRc, $(if ($ok) { 'PASS' } else { 'FAIL' })) -ForegroundColor $(if ($ok) { 'Green' } else { 'Red' })
  if (-not $ok) { $out | Write-Host }
  return $ok
}
$target = "TARGET_PG_USER=loader`nTARGET_PG_PASSWORD=TargetSecret456`n"
$oraOk  = "ORACLE_USER=el_user`nORACLE_PASSWORD=SuperSecret123`n" + $target
$oraMissingUser = "ORACLE_PASSWORD=SuperSecret123`n" + $target
$questOnly = "MSSQL_USER_QUEST=q_user`nMSSQL_PASSWORD_QUEST=QuestSecret789`n" + $target
Write-Host ''; Write-Host '--- env check self-test ---' -ForegroundColor Cyan
$t1 = Run-Scenario 'oracle_ok'      $oraOk          @{ CI_JOB_NAME = 'oracle_extract_load' }    0 @('passed for oracle_extract_load')
$t2 = Run-Scenario 'oracle_no_user' $oraMissingUser @{ CI_JOB_NAME = 'oracle_extract_load' }    1 @('ORACLE_USER is missing')
$t3 = Run-Scenario 'no_env_file'    ''              @{ CI_JOB_NAME = 'oracle_extract_load' }    1 @('EL env file not usable')
$t4 = Run-Scenario 'sqlsrv_missing' $oraOk          @{ CI_JOB_NAME = 'sqlserver_extract_load' } 1 @('MSSQL_USER is missing', 'MSSQL_PASSWORD is missing')
$t5 = Run-Scenario 'profile_only'   $questOnly      @{ CI_JOB_NAME = 'sqlserver_extract_load' } 0 @('only profile variants', 'passed')
$t6 = Run-Scenario 'bypass'         $oraMissingUser @{ CI_JOB_NAME = 'oracle_extract_load'; EL_ENV_CHECK = 'off' } 0 @('skipped')
$logOk = Test-Path (Join-Path $testRoot 'oracle_no_user\logs\oracle_extract_load.log')
Write-Host "  error written to job log: $logOk" -ForegroundColor $(if ($logOk) { 'Green' } else { 'Red' })
foreach ($k in $touch) { [Environment]::SetEnvironmentVariable($k, $saved[$k], 'Process') }
Remove-Item $testRoot -Recurse -Force -ErrorAction SilentlyContinue
if (-not ($t1 -and $t2 -and $t3 -and $t4 -and $t5 -and $t6 -and $logOk)) { Revert 'env check self-test failed'; return }

# ---------- 5. validate CI diff + YAML (only .el_job script changed) ----------
$num = (G diff --numstat -- $CiRel) -join ''
if ($num -ne "2`t0`t$CiRel") { Revert "unexpected numstat: '$num' (expected 2 added, 0 deleted)"; return }
$chk = Join-Path $env:TEMP 'check_ci_envcheck.py'
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
if set(a) != set(b): errs.append("top-level keys changed: %s" % (set(a) ^ set(b)))
for k in a:
    if k != ".el_job" and a[k] != b.get(k): errs.append("changed: " + k)
ea, eb = a[".el_job"], b[".el_job"]
changed = {k for k in set(ea) | set(eb) if ea.get(k) != eb.get(k)}
if changed != {"script"}: errs.append(".el_job changed keys: %s" % changed)
s = "\n".join(x if isinstance(x, str) else str(x) for x in (eb.get("script") or []))
ci, ri = s.find("scripts/el_env_check.py"), s.find("uv run python @elArgs")
if ci < 0 or ri < 0 or ci > ri: errs.append("env check not placed before the loader")
print("YAML CHECK: " + ("OK" if not errs else "; ".join(errs)))
sys.exit(0 if not errs else 3)
'@ | Set-Content -Path $chk -Encoding ASCII
Write-Host ''; Write-Host '--- YAML validation ---' -ForegroundColor Cyan
& $py -I $chk $orig $CiPath 2>&1 | ForEach-Object { Write-Host "  $_" }
if ($LASTEXITCODE -ne 0) { Revert 'YAML validation failed'; return }
Write-Host ''; Write-Host '--- DIFF .gitlab-ci.yml ---' -ForegroundColor Cyan
G diff -- $CiRel | Write-Host

# ---------- 6. commit + push (ci.skip) ----------
G add -- $CiRel $PyRel | Out-Null
$staged = (G diff --cached --name-status | Sort-Object) -join '|'
if ($staged -ne "A`t$PyRel|M`t$CiRel") { Revert "unexpected staged files: $staged"; return }
$c = G commit -m 'Add pre-flight env check to E&L jobs' -m 'New scripts/el_env_check.py runs in .el_job right before the loader. It loads the EL env file the same way the loaders do and fails fast with a clear message when the file is not usable, a required source setting (e.g. ORACLE_USER) is missing or empty, or the load target login/endpoint cannot be resolved. Values are never printed; errors are also written to the job log so el_health_check reports them. Base settings missing with only profile variants present is a warning, not a failure. EL_ENV_CHECK=off bypasses the check. Refs DI-4662.'
if ($LASTEXITCODE -ne 0) { $c | Select-Object -Last 40 | Write-Host; Revert 'commit failed'; return }
$p = G push -o ci.skip origin $Br
$p | Write-Host
if ($LASTEXITCODE -ne 0) { Write-Host 'PUSH FAILED - commit kept locally; send me a photo.' -ForegroundColor Red; return }

Write-Host ''
Write-Host '================ DONE ================' -ForegroundColor Green
Write-Host "Branch : $Br @ $((G rev-parse --short=8 HEAD) -join '')"
G log --oneline 'origin/development..HEAD' | ForEach-Object { Write-Host "  $_" }
}
