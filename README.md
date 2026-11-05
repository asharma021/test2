& {
$repo = 'C:\Users\SharmaAashishR\oa-business-layer'
Set-Location $repo
$tmp = 'tmp/prod-mirror-env-file'
$orig = (git rev-parse --abbrev-ref HEAD).Trim()
function Undo([string]$msg) {
  Write-Host "FAILED: $msg" -ForegroundColor Red
  git reset --hard --quiet 2>$null
  git switch --quiet $orig 2>$null
  git branch -D $tmp 2>$null | Out-Null
  Write-Host "Reverted: back on '$(git rev-parse --abbrev-ref HEAD)'. Nothing pushed." -ForegroundColor Yellow
}
if (git status --porcelain --untracked-files=no) { Write-Host "Uncommitted changes - nothing done." -ForegroundColor Red; return }
git fetch origin --prune 2>$null
git branch -D $tmp 2>$null | Out-Null
git switch --quiet -c $tmp origin/development
if ($LASTEXITCODE -ne 0) { Write-Host "Could not create temp branch - nothing done." -ForegroundColor Red; return }
$cp = Join-Path $repo '.gitlab-ci.yml'
$origYml = Join-Path $env:TEMP 'oa_ci_orig6.yml'
Copy-Item $cp $origYml -Force
$py = Join-Path $env:TEMP 'oa_edit_mirror_env.py'
@'
import sys, yaml
p, orig = sys.argv[1], sys.argv[2]
class L(yaml.SafeLoader): pass
L.add_constructor('!reference', lambda l, n: ['!reference'] + l.construct_sequence(n))
def load(f): return list(yaml.load_all(open(f, encoding='utf-8-sig'), Loader=L))
o = load(orig)
path = o[-1]['postgres_extract_load']['variables']['EL_ENV_FILE']
if "'" in path: sys.exit('unexpected quote in path')
pm = o[-1].get('prod_mirror')
if pm is None: sys.exit('prod_mirror job not found')
if isinstance(pm.get('variables'), dict) and 'EL_ENV_FILE' in pm['variables']: sys.exit('prod_mirror already sets EL_ENV_FILE')
raw = open(p, 'rb').read().decode('utf-8')
bom = raw.startswith('\ufeff')
if bom: raw = raw[1:]
nl = '\r\n' if '\r\n' in raw else '\n'
lines = raw.split(nl)
s = [i for i, l in enumerate(lines) if l == 'prod_mirror:']
if len(s) != 1: sys.exit(f"'prod_mirror:' found {len(s)} times")
s = s[0]; e = s + 1
while e < len(lines) and (lines[e][:1] in (' ', '\t') or lines[e] == ''): e += 1
entry = "    EL_ENV_FILE: '" + path + "'"
v = [i for i in range(s + 1, e) if lines[i] == '  variables:']
if len(v) == 1:
    lines.insert(v[0] + 1, entry)
else:
    x = [i for i in range(s + 1, e) if lines[i].startswith('  extends:')]
    if len(x) != 1: print('\n'.join(lines[s:e])); sys.exit('could not find where to add variables - send me this output')
    lines[x[0] + 1:x[0] + 1] = ['  variables:', entry]
open(p, 'wb').write((('\ufeff' if bom else '') + nl.join(lines)).encode('utf-8'))
n = load(p)
assert n[-1]['prod_mirror']['variables']['EL_ENV_FILE'] == path, 'value not set'
print('OK prod_mirror EL_ENV_FILE =', path)
del n[-1]['prod_mirror']['variables']['EL_ENV_FILE']
if not n[-1]['prod_mirror']['variables'] and 'variables' not in pm: del n[-1]['prod_mirror']['variables']
assert n == o, 'something else in the file changed'
print('OK nothing else changed')
'@ | Set-Content -Path $py -Encoding UTF8
uv run --no-sync python $py $cp $origYml
if ($LASTEXITCODE -ne 0) { Undo "edit/validation failed - see message above"; return }
Write-Host "`n===== DIFF =====" -ForegroundColor Cyan
git --no-pager diff
$msgFile = Join-Path $env:TEMP 'oa_mirror_env_msg.txt'
$msg = "Point prod_mirror at the extract/load env file`n`nprod_mirror failed with 'Missing required env vars: MIRROR_PG_*' because no`nEL_ENV_FILE was set for the job. It now uses the same env file path as`npostgres_extract_load, which contains the MIRROR_PG_* settings.`n"
[IO.File]::WriteAllText($msgFile, $msg, (New-Object Text.UTF8Encoding($false)))
git add .gitlab-ci.yml
git commit --quiet -F $msgFile
if ($LASTEXITCODE -ne 0) { Undo "commit failed (hook?) - see output above"; return }
Write-Host "Pushing directly to development ..." -ForegroundColor Cyan
git push origin HEAD:development
if ($LASTEXITCODE -eq 0) {
  Write-Host "DONE: pushed to development." -ForegroundColor Green
} else {
  Write-Host "Direct push rejected - opening a small MR instead." -ForegroundColor Yellow
  $br = 'fix/prod-mirror-env-file'
  git branch -m $tmp $br; $tmp = $br
  git push -u origin $br -o ci.skip -o merge_request.create -o merge_request.target=development -o "merge_request.title=Point prod_mirror at the extract/load env file"
  if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - send me the error." -ForegroundColor Red; return }
  Write-Host "MR created - merge it into development." -ForegroundColor Green
}
git switch --quiet $orig
git branch -D $tmp 2>$null | Out-Null
Write-Host "Back on $(git rev-parse --abbrev-ref HEAD)." -ForegroundColor Green
}
