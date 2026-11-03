& {
$repo = 'C:\Users\SharmaAashishR\oa-business-layer'
Set-Location $repo
$branch = 'ci/env-setup-auto-on-development'
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
$origYml = Join-Path $env:TEMP 'oa_ci_orig2.yml'
Copy-Item $cp $origYml -Force
$py = Join-Path $env:TEMP 'oa_edit_envsetup.py'
@'
import sys, yaml
p, orig = sys.argv[1], sys.argv[2]
raw = open(p, 'rb').read().decode('utf-8')
bom = raw.startswith('\ufeff')
if bom: raw = raw[1:]
nl = '\r\n' if '\r\n' in raw else '\n'
lines = raw.split(nl)
s = [i for i, l in enumerate(lines) if l == 'env_setup:']
if len(s) != 1: sys.exit(f"'env_setup:' found {len(s)} times")
s = s[0]; e = s + 1
while e < len(lines) and (lines[e][:1] in (' ', '\t') or lines[e] == ''): e += 1
hits = [i for i in range(s + 1, e) if '!reference' in lines[i] and '.manual_on_development' in lines[i]]
if len(hits) != 1:
    print('\n'.join(lines[s:e])); sys.exit(f'manual_on_development reference found {len(hits)} times in env_setup')
print('Removing line', hits[0] + 1, ':', lines[hits[0]].strip())
del lines[hits[0]]
open(p, 'wb').write((('\ufeff' if bom else '') + nl.join(lines)).encode('utf-8'))
class L(yaml.SafeLoader): pass
L.add_constructor('!reference', lambda l, n: ['!reference'] + l.construct_sequence(n))
def load(f): return list(yaml.load_all(open(f, encoding='utf-8-sig'), Loader=L))
n, o = load(p), load(orig)
ref = ['!reference', '.manual_on_development', 'rules']
assert ref not in n[-1]['env_setup']['rules'], 'still present'
assert [r for r in o[-1]['env_setup']['rules'] if r != ref] == n[-1]['env_setup']['rules'], 'other env_setup rules changed'
n[-1]['env_setup'].pop('rules'); o[-1]['env_setup'].pop('rules')
assert n == o, 'something else in the file changed'
print('OK env_setup no longer manual on development pushes; nothing else changed')
'@ | Set-Content -Path $py -Encoding UTF8
uv run --no-sync python $py $cp $origYml
if ($LASTEXITCODE -ne 0) { Undo "edit/validation failed - see message above"; return }
Write-Host "`n===== DIFF =====" -ForegroundColor Cyan
git --no-pager diff
$msgFile = Join-Path $env:TEMP 'oa_envsetup_msg.txt'
$msg = "Run env_setup automatically after merges into development`n`nenv_setup was manual on development push pipelines, so every merge left the`npipeline Blocked until someone played it. It now runs automatically, followed`nby dbt_compile (read-only against DEV), so each merge is validated right away.`nThe run-stage jobs (seed, run, test) are unchanged and stay skipped on pushes.`n"
[IO.File]::WriteAllText($msgFile, $msg, (New-Object Text.UTF8Encoding($false)))
git add .gitlab-ci.yml
git commit --quiet -F $msgFile
if ($LASTEXITCODE -ne 0) { Undo "commit failed (hook?) - see output above"; return }
git push -u origin $branch -o ci.skip -o merge_request.create -o merge_request.target=development -o "merge_request.title=Run env_setup automatically after merges into development"
if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - commit kept locally on $branch; send me the error." -ForegroundColor Red; return }
Write-Host "DONE: $(git log -1 --format='%h %s') pushed; MR into development created." -ForegroundColor Green
git switch --quiet $orig
Write-Host "Back on $(git rev-parse --abbrev-ref HEAD)." -ForegroundColor Green
}
