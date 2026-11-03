& {
$repo = 'C:\Users\SharmaAashishR\oa-business-layer'
Set-Location $repo
$branch = 'fix/dbt-evaluate-status'
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
$origYml = Join-Path $env:TEMP 'oa_ci_orig4.yml'
Copy-Item $cp $origYml -Force
$py = Join-Path $env:TEMP 'oa_edit_evaluate2.py'
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
if any('Evaluator build step finished' in lines[i] for i in range(s, e)): sys.exit('already patched')
hits = [i for i in range(s + 1, e) if 'Evaluator found issues' in lines[i]]
if len(hits) != 1: print('\n'.join(lines[s:e])); sys.exit(f"'Evaluator found issues' line found {len(hits)} times")
h = hits[0]
if lines[h].lstrip().startswith('-'): print('\n'.join(lines[s:e])); sys.exit('unexpected layout - send me this output')
ind = lines[h][:len(lines[h]) - len(lines[h].lstrip())]
new_txt = [
    "# The dbt line pipes stderr (2>&1), which leaves PowerShell's success flag (`$?)",
    "# false even when dbt exits 0; the runner then fails the job before the report",
    "# step runs. End the step with a command that succeeds to reset it.",
    'Write-Host "Evaluator build step finished."',
]
lines[h + 1:h + 1] = [ind + t for t in new_txt]
open(p, 'wb').write((('\ufeff' if bom else '') + nl.join(lines)).encode('utf-8'))
class L(yaml.SafeLoader): pass
L.add_constructor('!reference', lambda l, n: ['!reference'] + l.construct_sequence(n))
def load(f): return list(yaml.load_all(open(f, encoding='utf-8-sig'), Loader=L))
n, o = load(p), load(orig)
ns, os_ = n[-1]['dbt_evaluate']['script'], o[-1]['dbt_evaluate']['script']
assert len(ns) == len(os_), 'number of script steps changed'
idx = [i for i, x in enumerate(ns) if isinstance(x, str) and 'Evaluator build step finished' in x]
assert len(idx) == 1, 'insert not found in exactly one step'
i = idx[0]
strip = set(new_txt)
assert [l for l in ns[i].split('\n') if l.strip() not in strip] == os_[i].split('\n'), 'that step changed beyond the insert'
assert ns[i].rstrip('\n').split('\n')[-1].strip() == 'Write-Host "Evaluator build step finished."', 'new line is not the last line of the step'
assert [x for j, x in enumerate(ns) if j != i] == [x for j, x in enumerate(os_) if j != i], 'other steps changed'
n[-1]['dbt_evaluate'].pop('script'); o[-1]['dbt_evaluate'].pop('script')
assert n == o, 'something else in the file changed'
print('OK dbt_evaluate step', i + 1, 'now ends with a successful command; nothing else changed')
'@ | Set-Content -Path $py -Encoding UTF8
uv run --no-sync python $py $cp $origYml
if ($LASTEXITCODE -ne 0) { Undo "edit/validation failed - see message above"; return }
Write-Host "`n===== DIFF =====" -ForegroundColor Cyan
git --no-pager diff
$msgFile = Join-Path $env:TEMP 'oa_evaluate2_msg.txt'
$msg = "Reset PowerShell success flag at the end of the dbt_evaluate build step`n`nThe evaluator build pipes stderr (2>&1 | Tee-Object), which leaves `$? false`neven when dbt exits 0. The runner checks `$? after each script step, so it failed`nthe job before the Code Quality report step ran. The step now ends with a`ncommand that succeeds; dbt's exit code and the report step still decide success.`n"
[IO.File]::WriteAllText($msgFile, $msg, (New-Object Text.UTF8Encoding($false)))
git add .gitlab-ci.yml
git commit --quiet -F $msgFile
if ($LASTEXITCODE -ne 0) { Undo "commit failed (hook?) - see output above"; return }
git push -u origin $branch -o ci.skip -o merge_request.create -o merge_request.target=development -o "merge_request.title=Fix dbt_evaluate failing after a successful evaluator build"
if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - commit kept locally on $branch; send me the error." -ForegroundColor Red; return }
Write-Host "DONE: $(git log -1 --format='%h %s') pushed; MR into development created." -ForegroundColor Green
git switch --quiet $orig
Write-Host "Back on $(git rev-parse --abbrev-ref HEAD)." -ForegroundColor Green
}
