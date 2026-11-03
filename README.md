& {
$repo = 'C:\Users\SharmaAashishR\oa-business-layer'
Set-Location $repo
$branch = 'fix/dbt-evaluate-stderr'
$orig = (git rev-parse --abbrev-ref HEAD).Trim()
function Undo([string]$msg) {
  Write-Host "FAILED: $msg" -ForegroundColor Red
  git reset --hard --quiet 2>$null
  git switch --quiet $orig 2>$null
  if ((git rev-parse --abbrev-ref HEAD).Trim() -eq $orig) { git branch -D $branch 2>$null | Out-Null; Write-Host "Reverted: back on '$orig', '$branch' removed. Nothing pushed." -ForegroundColor Yellow }
}
if (git status --porcelain --untracked-files=no) { Write-Host "Uncommitted changes - nothing done." -ForegroundColor Red; return }
git fetch origin --prune 2>$null
if (git branch --list $branch) { Write-Host "Local $branch exists - nothing done." -ForegroundColor Red; return }
if (git ls-remote --heads origin $branch) { Write-Host "Remote $branch exists - nothing done." -ForegroundColor Red; return }
git switch --quiet -c $branch origin/development
if ($LASTEXITCODE -ne 0) { Write-Host "Could not create branch - nothing done." -ForegroundColor Red; return }
$cp = Join-Path $repo '.gitlab-ci.yml'
$origYml = Join-Path $env:TEMP 'oa_ci_orig3.yml'
Copy-Item $cp $origYml -Force
$py = Join-Path $env:TEMP 'oa_edit_evaluate.py'
@'
import sys, yaml
p, orig = sys.argv[1], sys.argv[2]
raw = open(p, 'rb').read().decode('utf-8')
bom = raw.startswith('\ufeff')
if bom: raw = raw[1:]
nl = '\r\n' if '\r\n' in raw else '\n'
lines = raw.split(nl)
s = [i for i, l in enumerate(lines) if l == 'dbt_evaluate:']
if len(s) != 1: sys.exit(f"'dbt_evaluate:' found {len(s)} times")
s = s[0]; e = s + 1
while e < len(lines) and (lines[e][:1] in (' ', '\t') or lines[e] == ''): e += 1
hits = [i for i in range(s + 1, e) if lines[i].lstrip().startswith('Write-Host "=== Running dbt-project-evaluator')]
if len(hits) != 1: print('\n'.join(lines[s:e])); sys.exit(f'evaluator Write-Host line found {len(hits)} times')
h = hits[0]
if any('ErrorActionPreference' in lines[i] for i in range(s, e)): sys.exit('already patched')
ind = lines[h][:len(lines[h]) - len(lines[h].lstrip())]
new = [
    ind + "# dbt/Python on this Windows runner sometimes writes a harmless shutdown",
    ind + "# traceback (WinError 6) to stderr after a successful build. With the runner's",
    ind + "# ErrorActionPreference=Stop that aborted the step before the report ran.",
    ind + "# Exit codes still decide success.",
    ind + "$ErrorActionPreference = 'Continue'",
]
lines[h:h] = new
open(p, 'wb').write((('\ufeff' if bom else '') + nl.join(lines)).encode('utf-8'))
class L(yaml.SafeLoader): pass
L.add_constructor('!reference', lambda l, n: ['!reference'] + l.construct_sequence(n))
def load(f): return list(yaml.load_all(open(f, encoding='utf-8-sig'), Loader=L))
n, o = load(p), load(orig)
ns, os_ = n[-1]['dbt_evaluate']['script'], o[-1]['dbt_evaluate']['script']
assert len(ns) == len(os_) and ns[1:] == os_[1:], 'other script entries changed'
assert "$ErrorActionPreference = 'Continue'" in ns[0], 'line not in script'
assert ns[0].replace(nl.join(x[len(ind):] for x in new) + '\n', '', 1) == os_[0] or ns[0].replace('\n'.join(x[len(ind):] for x in new) + '\n', '', 1) == os_[0], 'evaluator script changed beyond the insert'
n[-1]['dbt_evaluate'].pop('script'); o[-1]['dbt_evaluate'].pop('script')
assert n == o, 'something else in the file changed'
print('OK dbt_evaluate patched; nothing else changed')
'@ | Set-Content -Path $py -Encoding UTF8
uv run --no-sync python $py $cp $origYml
if ($LASTEXITCODE -ne 0) { Undo "edit/validation failed - see message above"; return }
Write-Host "`n===== DIFF =====" -ForegroundColor Cyan
git --no-pager diff
$msgFile = Join-Path $env:TEMP 'oa_evaluate_msg.txt'
$msg = "Keep dbt_evaluate from failing on harmless stderr output`n`nAfter a successful evaluator build, dbt/Python on the Windows runner sometimes`nwrites a shutdown traceback (WinError 6) to stderr. With the runner's`nErrorActionPreference=Stop, that aborted the step before the Code Quality`nreport was generated, so dbt_evaluate failed intermittently. The step now uses`nErrorActionPreference=Continue; exit codes still decide success.`n"
[IO.File]::WriteAllText($msgFile, $msg, (New-Object Text.UTF8Encoding($false)))
git add .gitlab-ci.yml
git commit --quiet -F $msgFile
if ($LASTEXITCODE -ne 0) { Undo "commit failed (hook?) - see output above"; return }
git push -u origin $branch -o ci.skip -o merge_request.create -o merge_request.target=development -o "merge_request.title=Keep dbt_evaluate from failing on harmless stderr output"
if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - commit kept locally on $branch; send me the error." -ForegroundColor Red; return }
Write-Host "DONE: $(git log -1 --format='%h %s') pushed; MR into development created." -ForegroundColor Green
git switch --quiet $orig
Write-Host "Back on $(git rev-parse --abbrev-ref HEAD)." -ForegroundColor Green
}
