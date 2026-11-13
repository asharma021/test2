& {
# ===== READ-ONLY: env variable inventory for DI-4662 =====
$ErrorActionPreference = 'Continue'
$Repo = 'C:\Users\SharmaAashishR\oa-business-layer'
$Out  = Join-Path $env:TEMP 'env_integrity_inspect.txt'
if (-not (Test-Path (Join-Path $Repo '.git'))) { Write-Host 'ABORT: not a git repo' -ForegroundColor Red; return }
Set-Location $Repo
function G { @(& git --no-pager -c core.quotepath=off @args 2>&1 | ForEach-Object { "$_" }) }
$statusBefore = G status --porcelain --untracked-files=all
$headBefore   = (G rev-parse HEAD) -join ''
if (-not (Test-Path '.venv\Scripts\python.exe')) { Write-Host 'ABORT: .venv python not found' -ForegroundColor Red; return }
$py = (Resolve-Path '.venv\Scripts\python.exe').Path

$pyFile = Join-Path $env:TEMP 'env_inspect.py'
@'
import re, sys, subprocess
from pathlib import Path
from collections import defaultdict

root = Path(sys.argv[1])
files = subprocess.run(["git", "ls-files"], capture_output=True, text=True, cwd=root).stdout.splitlines()

print("=" * 100); print("1. ENV ACCESS IN PYTHON (src/, scripts/)"); print("=" * 100)
acc = re.compile(r"os\.environ|getenv|environ\.get|load_dotenv|dotenv_values|\benviron\b")
lit = re.compile(r"""(?:environ\[|getenv\(|environ\.get\(|\.get\(|\[)\s*f?["']([A-Z][A-Z0-9_{}]*)["']""")
names = defaultdict(set)
for f in files:
    if not f.endswith(".py") or not (f.startswith("src/") or f.startswith("scripts/")):
        continue
    for n, line in enumerate((root / f).read_text(encoding="utf-8", errors="replace").splitlines(), 1):
        if acc.search(line) or re.search(r"""["'][A-Z][A-Z0-9]*_[A-Z0-9_]*(HOST|PORT|USER|PASSWORD|PASS|DB|DATABASE|DSN|SERVICE|SID|SERVER|SCHEMA|URL|DRIVER)["']""", line):
            print("%s:%d: %s" % (f, n, line.strip()))
            for m in lit.finditer(line):
                names[m.group(1)].add(f)

print(); print("=" * 100); print("2. VARIABLE NAMES / PATTERNS FOUND"); print("=" * 100)
for k in sorted(names):
    print("  %-40s %s" % (k, ", ".join(sorted(names[k]))))

print(); print("=" * 100); print("3. ENV TEMPLATES / EXAMPLES IN REPO"); print("=" * 100)
for f in files:
    low = f.lower()
    if re.search(r"(^|/)\.?env(\.|$)|\.env\.|env\.example|env_template|example\.env|\.env$", low):
        print("  " + f)
        for line in (root / f).read_text(encoding="utf-8", errors="replace").splitlines():
            key = line.split("=", 1)[0].strip()
            if key and not key.startswith("#"):
                print("      " + key + ("=<set>" if "=" in line and line.split("=", 1)[1].strip() else "="))

print(); print("=" * 100); print("4. CI: .el_job / .dbt_env BLOCKS + env file references"); print("=" * 100)
ci = (root / ".gitlab-ci.yml").read_text(encoding="utf-8", errors="replace").splitlines()
def block(name):
    out, on = [], False
    for n, line in enumerate(ci, 1):
        if re.match(r"^%s:\s*(#.*)?$" % re.escape(name), line):
            on = True
        elif on and re.match(r"^[^\s#]", line):
            break
        if on:
            out.append("%5d: %s" % (n, line))
    return out
for b in (".dbt_env", ".el_job", ".el_upstream", "oracle_extract_load", "sqlserver_extract_load",
          "postgres_extract_load", "informix_extract_load", "prod_mirror"):
    print(); print("--- %s ---" % b)
    for l in block(b):
        print(l)
print(); print("--- lines mentioning extract_load.env / env loading ---")
for n, line in enumerate(ci, 1):
    if re.search(r"extract_load\.env|Get-Content|SetEnvironmentVariable|\$env:[A-Z_]+\s*=", line):
        print("%5d: %s" % (n, line))
'@ | Set-Content -Path $pyFile -Encoding ASCII

$log = New-Object System.Collections.Generic.List[string]
& $py -I $pyFile $Repo 2>&1 | ForEach-Object { $log.Add("$_") }
$same = (($statusBefore -join "`n") -eq ((G status --porcelain --untracked-files=all) -join "`n")) -and ($headBefore -eq ((G rev-parse HEAD) -join ''))
$log.Add(''); $log.Add("Working tree + HEAD unchanged: $same")
$log | Set-Content -Path $Out -Encoding UTF8
Write-Host "Done. $($log.Count) lines -> $Out" -ForegroundColor Cyan
Write-Host "Working tree + HEAD unchanged: $same" -ForegroundColor Green
}
