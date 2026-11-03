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
if any('ErrorActionPreference' in lines[i] for i in range(s, e)): sys.exit('already patched')
hits = [i for i in range(s + 1, e) if 'uv run dbt build' in lines[i] and 'dbt_project_evaluator' in lines[i]]
if len(hits) != 1: print('\n'.join(lines[s:e])); sys.exit(f'dbt build line found {len(hits)} times')
h = hits[0]
if lines[h].lstrip().startswith('-'):
    print('\n'.join(lines[s:e])); sys.exit('dbt build line is a single-line list item, not inside a | block - send me this output')
ind = lines[h][:len(lines[h]) - len(lines[h].lstrip())]
new_txt = [
    "# dbt/Python on this Windows runner sometimes writes a harmless shutdown",
    "# traceback (WinError 6) to stderr after a successful build. With the runner's",
    "# ErrorActionPreference=Stop that aborted the job before the report ran.",
    "# Exit codes still decide success.",
    "$ErrorActionPreference = 'Continue'",
]
lines[h:h] = [ind + t for t in new_txt]
open(p, 'wb').write((('\ufeff' if bom else '') + nl.join(lines)).encode('utf-8'))
class L(yaml.SafeLoader): pass
L.add_constructor('!reference', lambda l, n: ['!reference'] + l.construct_sequence(n))
def load(f): return list(yaml.load_all(open(f, encoding='utf-8-sig'), Loader=L))
n, o = load(p), load(orig)
ns, os_ = n[-1]['dbt_evaluate']['script'], o[-1]['dbt_evaluate']['script']
assert len(ns) == len(os_), 'number of script steps changed'
idx = [i for i, x in enumerate(ns) if isinstance(x, str) and "$ErrorActionPreference = 'Continue'" in x]
assert len(idx) == 1, 'insert not found in exactly one script step'
i = idx[0]
strip = set(new_txt)
assert [l for l in ns[i].split('\n') if l.strip() not in strip] == os_[i].split('\n'), 'that step changed beyond the insert'
assert [x for j, x in enumerate(ns) if j != i] == [x for j, x in enumerate(os_) if j != i], 'other script steps changed'
n[-1]['dbt_evaluate'].pop('script'); o[-1]['dbt_evaluate'].pop('script')
assert n == o, 'something else in the file changed'
print('OK dbt_evaluate patched (script step', i + 1, 'of', len(ns), '); nothing else changed')
'@ | Set-Content -Path $py -Encoding UTF8
uv run --no-sync python $py $cp $origYml
if ($LASTEXITCODE -ne 0) { Undo "edit/validation failed - see message above"; return }
Write-Host "`n===== DIFF =====" -ForegroundColor Cyan
git --no-pager diff
$msgFile = Join-Path $env:TEMP 'oa_evaluate_msg.txt'
$msg = "Keep dbt_evaluate from failing on harmless stderr output`n`nAfter a successful evaluator build, dbt/Python on the Windows runner sometimes`nwrites a shutdown traceback (WinError 6) to stderr. With the runner's`nErrorActionPreference=Stop, that aborted the job before the Code Quality`nreport was generated, so dbt_evaluate failed intermittently. The job now uses`nErrorActionPreference=Continue; exit codes still decide success.`n"
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
