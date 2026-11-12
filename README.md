& {
# ===== READ-ONLY: E&L failure / notification inventory (DI-4659) - v2 =====
$ErrorActionPreference = 'Continue'; $env:GIT_TERMINAL_PROMPT = '0'
$Repo = 'C:\Users\SharmaAashishR\oa-business-layer'
$Refs = @('HEAD', 'origin/production')
$Out  = Join-Path $env:TEMP 'el_notify_inventory.txt'
if (-not (Test-Path (Join-Path $Repo '.git'))) { Write-Host 'ABORT: not a git repo' -ForegroundColor Red; return }
Set-Location $Repo
function G { @(& git --no-pager -c core.quotepath=off @args 2>&1 | ForEach-Object { "$_" }) }
$statusBefore = G status --porcelain --untracked-files=all
$headBefore   = (G rev-parse HEAD) -join ''

$py = $null
if (Test-Path '.venv\Scripts\python.exe') { $py = (Resolve-Path '.venv\Scripts\python.exe').Path }
elseif (Get-Command py -ErrorAction SilentlyContinue) { $py = 'py' }
elseif (Get-Command python -ErrorAction SilentlyContinue) { $py = 'python' }
if (-not $py) { Write-Host 'ABORT: no Python found' -ForegroundColor Red; return }
$pyFile = Join-Path $env:TEMP 'ci_inventory.py'
@'
import sys, subprocess, yaml
class L(yaml.SafeLoader): pass
def unk(loader, suffix, node):
    if isinstance(node, yaml.ScalarNode):   return "!%s %s" % (suffix, loader.construct_scalar(node))
    if isinstance(node, yaml.SequenceNode): return {"!" + suffix: loader.construct_sequence(node, deep=True)}
    return {"!" + suffix: loader.construct_mapping(node, deep=True)}
L.add_multi_constructor("!", unk)
RESERVED = {"stages","variables","default","include","workflow","image","services","before_script","after_script","cache","spec"}
for ref in sys.argv[1:]:
    p = subprocess.run(["git","show","%s:.gitlab-ci.yml" % ref], capture_output=True, text=True, encoding="utf-8")
    docs = [x for x in yaml.load_all(p.stdout, Loader=L) if isinstance(x, dict)]
    d = {}
    spec = None
    for x in docs:
        if "spec" in x and len(x) == 1: spec = x["spec"]
        else: d.update(x)
    print("=" * 100); print("REF %s  (documents: %d)" % (ref, len(docs))); print("=" * 100)
    if spec:
        print("spec.inputs:")
        for k, v in (spec.get("inputs") or {}).items():
            v = v or {}
            print("   %-28s type=%s default=%s options=%s" % (k, v.get("type"), v.get("default"), v.get("options")))
    print("stages  :", d.get("stages"))
    print("include :", d.get("include"))
    print("workflow:", d.get("workflow"))
    print("default :", d.get("default"))
    jobs = {k: v for k, v in d.items() if k not in RESERVED and isinstance(v, dict)}
    def resolved(name, seen=None):
        seen = seen if seen is not None else set(); j = jobs.get(name, {}); out = {}
        ext = j.get("extends", []); ext = [ext] if isinstance(ext, str) else (ext or [])
        for e in ext:
            if e not in seen: seen.add(e); out.update(resolved(e, seen))
        out.update(j); return out
    for name in jobs:
        r = resolved(name); rules = r.get("rules")
        rw = sorted({str(x.get("when")) for x in rules if isinstance(x, dict) and "when" in x}) if isinstance(rules, list) else []
        art = r.get("artifacts") or {}
        print("\n[%s]%s stage=%s extends=%s" % (name, " (template)" if name.startswith(".") else "", r.get("stage"), jobs[name].get("extends")))
        print("   allow_failure=%s when=%s rule_whens=%s timeout=%s retry=%s" % (r.get("allow_failure"), r.get("when"), rw, r.get("timeout"), r.get("retry")))
        print("   needs=%s after_script=%s tags=%s" % (r.get("needs"), "yes" if r.get("after_script") else "no", r.get("tags")))
        if art: print("   artifacts: when=%s paths=%s expire_in=%s" % (art.get("when"), art.get("paths"), art.get("expire_in")))
'@ | Set-Content -Path $pyFile -Encoding ASCII

$log = New-Object System.Collections.Generic.List[string]
& $py -I $pyFile @Refs 2>&1 | ForEach-Object { $log.Add("$_") }

$same = (($statusBefore -join "`n") -eq ((G status --porcelain --untracked-files=all) -join "`n")) -and ($headBefore -eq ((G rev-parse HEAD) -join ''))
$log.Add(''); $log.Add("Working tree + HEAD unchanged: $same")
$log | Set-Content -Path $Out -Encoding UTF8
$log | Write-Host
Write-Host ''
Write-Host "Saved: $Out ($($log.Count) lines)" -ForegroundColor Cyan
}
