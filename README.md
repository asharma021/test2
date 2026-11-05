& {
$repo = 'C:\Users\SharmaAashishR\oa-business-layer'
Set-Location $repo
$tmp = 'tmp/cleanup-temp-ci'
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
$origYml = Join-Path $env:TEMP 'oa_ci_orig_cleanup.yml'
Copy-Item $cp $origYml -Force
$py = Join-Path $env:TEMP 'oa_cleanup.py'
@'
import sys, yaml
p, orig = sys.argv[1], sys.argv[2]
remove = ['.postgres_incremental_guard', 'postgres_extract_load_dev_incremental',
          'postgres_extract_load_ivv_incremental', 'postgres_extract_load_dev_verify',
          'postgres_extract_load_test', 'prod_mirror_dev_verify']
data = open(p, 'rb').read()
bom = data.startswith(b'\xef\xbb\xbf')
text = data.decode('utf-8-sig')
nl = '\r\n' if '\r\n' in text else '\n'
lines = text.split(nl)
spans = []
for name in remove:
    hits = [i for i, l in enumerate(lines) if l.rstrip() == name + ':']
    if len(hits) != 1: sys.exit(f'{name}: found {len(hits)} times')
    s = hits[0]
    while s > 0 and lines[s - 1].startswith('#'): s -= 1
    e = hits[0] + 1
    while e < len(lines) and (lines[e] == '' or lines[e][:1] in (' ', '\t')): e += 1
    spans.append((s, e, name))
for s, e, name in sorted(spans, reverse=True):
    print(f'Removing {name}: lines {s+1}-{e}')
    del lines[s:e]
new = nl.join(lines)
open(p, 'wb').write((b'\xef\xbb\xbf' if bom else b'') + new.encode('utf-8'))
for name in remove + ['tables_ivv_verify']:
    if name in new: sys.exit(f'"{name}" is still referenced somewhere in .gitlab-ci.yml')
class L(yaml.SafeLoader): pass
L.add_constructor('!reference', lambda l, n: ['!reference'] + l.construct_sequence(n))
def load(f): return list(yaml.load_all(open(f, encoding='utf-8-sig'), Loader=L))
n, o = load(p), load(orig)
assert len(n) == len(o) and n[:-1] == o[:-1], 'spec/inputs section changed'
nj, oj = n[-1], o[-1]
for name in remove: assert name in oj and name not in nj, name
rest = {k: v for k, v in oj.items() if k not in remove}
assert nj == rest, 'a remaining job changed'
print('OK removed', len(remove), 'jobs/templates; all', len(nj), 'remaining entries identical')
'@ | Set-Content -Path $py -Encoding UTF8
uv run --no-sync python $py $cp $origYml
if ($LASTEXITCODE -ne 0) { Undo "edit/validation failed - see message above"; return }
git rm --quiet src/extract_load/postgres/tables_ivv_verify.yml
if ($LASTEXITCODE -ne 0) { Undo "could not remove tables_ivv_verify.yml"; return }
$left = git grep -n 'tables_ivv_verify' -- . 2>$null
if ($left) { Write-Host "Still referenced:" -ForegroundColor Red; $left; Undo "tables_ivv_verify still referenced"; return }
Write-Host "`n===== CHANGES =====" -ForegroundColor Cyan
git add .gitlab-ci.yml
git --no-pager diff --cached --stat
$msgFile = Join-Path $env:TEMP 'oa_cleanup_msg.txt'
$msg = "Remove temporary DEV/IVV extract/load jobs and tables_ivv_verify.yml`n`nThese manual pipeline-testing jobs were used to validate PostgreSQL`nextract/load in DEV and IVV and the mirror connection. Production extract/load`nand the prod-to-lower-environment mirror now cover this, so they are removed:`n.postgres_incremental_guard, postgres_extract_load_dev_incremental,`npostgres_extract_load_ivv_incremental, postgres_extract_load_dev_verify,`npostgres_extract_load_test, prod_mirror_dev_verify, and the IVV-only table`nmapping tables_ivv_verify.yml. All other jobs are unchanged.`n"
[IO.File]::WriteAllText($msgFile, $msg, (New-Object Text.UTF8Encoding($false)))
git commit --quiet -F $msgFile
if ($LASTEXITCODE -ne 0) { Undo "commit failed (hook?) - see output above"; return }
Write-Host "Pushing directly to development ..." -ForegroundColor Cyan
git push origin HEAD:development
if ($LASTEXITCODE -eq 0) {
  Write-Host "DONE: pushed to development." -ForegroundColor Green
} else {
  Write-Host "Direct push rejected - opening a small MR instead." -ForegroundColor Yellow
  $br = 'chore/remove-temp-el-jobs'
  git branch -m $tmp $br; $tmp = $br
  git push -u origin $br -o ci.skip -o merge_request.create -o merge_request.target=development -o "merge_request.title=Remove temporary DEV/IVV extract/load jobs and tables_ivv_verify.yml"
  if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - send me the error." -ForegroundColor Red; return }
  Write-Host "MR created - merge it into development." -ForegroundColor Green
}
git switch --quiet $orig
git branch -D $tmp 2>$null | Out-Null
Write-Host "Back on $(git rev-parse --abbrev-ref HEAD)." -ForegroundColor Green
}
