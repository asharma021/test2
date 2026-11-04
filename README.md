& {
$repo = 'C:\Users\SharmaAashishR\oa-business-layer'
Set-Location $repo
$tmp = 'tmp/prod-postgres-auto'
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
$origYml = Join-Path $env:TEMP 'oa_ci_orig5.yml'
Copy-Item $cp $origYml -Force
$py = Join-Path $env:TEMP 'oa_edit_prodpg.py'
@'
import sys, yaml
p, orig = sys.argv[1], sys.argv[2]
raw = open(p, 'rb').read().decode('utf-8')
bom = raw.startswith('\ufeff')
if bom: raw = raw[1:]
nl = '\r\n' if '\r\n' in raw else '\n'
lines = raw.split(nl)
s = [i for i, l in enumerate(lines) if l == 'postgres_extract_load:']
if len(s) != 1: sys.exit(f"'postgres_extract_load:' found {len(s)} times")
s = s[0]; e = s + 1
while e < len(lines) and (lines[e][:1] in (' ', '\t') or lines[e] == ''): e += 1
r = [i for i in range(s + 1, e) if 'CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH' in lines[i]]
if len(r) != 1: sys.exit(f'default-branch rule found {len(r)} times')
r = r[0]
blk = lines[r:r + 5]
print('Current rule:'); print('\n'.join(blk))
ds = [i for i in range(r + 1, r + 5) if lines[i].strip() == 'DBT_SELECT: "myttb_cg_ref_identifier_type"']
wm = [i for i in range(r + 1, r + 5) if lines[i].strip() == 'when: manual']
if len(ds) != 1 or len(wm) != 1: sys.exit('expected DBT_SELECT default and when: manual in that rule - send me this output')
for i in sorted(ds + wm, reverse=True): del lines[i]
open(p, 'wb').write((('\ufeff' if bom else '') + nl.join(lines)).encode('utf-8'))
class L(yaml.SafeLoader): pass
L.add_constructor('!reference', lambda l, n: ['!reference'] + l.construct_sequence(n))
def load(f): return list(yaml.load_all(open(f, encoding='utf-8-sig'), Loader=L))
n, o = load(p), load(orig)
nr = [x for x in n[-1]['postgres_extract_load']['rules'] if 'CI_DEFAULT_BRANCH' in str(x.get('if', ''))][0]
assert nr == {'if': '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH', 'variables': {'DBT_TARGET': 'prod'}}, nr
print('OK new production rule:', nr)
orr = o[-1]['postgres_extract_load']['rules']; nrr = n[-1]['postgres_extract_load']['rules']
assert [x for x in orr if 'CI_DEFAULT_BRANCH' not in str(x.get('if', ''))] == [x for x in nrr if 'CI_DEFAULT_BRANCH' not in str(x.get('if', ''))], 'other rules changed'
n[-1]['postgres_extract_load'].pop('rules'); o[-1]['postgres_extract_load'].pop('rules')
assert n == o, 'something else in the file changed'
print('OK nothing else changed')
'@ | Set-Content -Path $py -Encoding UTF8
uv run --no-sync python $py $cp $origYml
if ($LASTEXITCODE -ne 0) { Undo "edit/validation failed - see message above"; return }
Write-Host "`n===== DIFF =====" -ForegroundColor Cyan
git --no-pager diff
$msgFile = Join-Path $env:TEMP 'oa_prodpg_msg.txt'
$msg = "Run PostgreSQL extract/load automatically on production for all tables`n`nRemoves the manual gate and the single-table default from the production rule`nof postgres_extract_load, so it loads every table in tables.yml and runs in the`nscheduled production pipeline like the other extract/loads. The prod target`n(oablmsdb / business_layer_prod) is unchanged.`n"
[IO.File]::WriteAllText($msgFile, $msg, (New-Object Text.UTF8Encoding($false)))
git add .gitlab-ci.yml
git commit --quiet -F $msgFile
if ($LASTEXITCODE -ne 0) { Undo "commit failed (hook?) - see output above"; return }
Write-Host "Pushing directly to development ..." -ForegroundColor Cyan
git push origin HEAD:development
if ($LASTEXITCODE -eq 0) {
  Write-Host "DONE: pushed to development. MR !17 now includes it." -ForegroundColor Green
} else {
  Write-Host "Direct push rejected - opening a small MR instead." -ForegroundColor Yellow
  $br = 'ci/prod-postgres-auto'
  git branch -m $tmp $br; $tmp = $br
  git push -u origin $br -o ci.skip -o merge_request.create -o merge_request.target=development -o "merge_request.title=Run PostgreSQL extract/load automatically on production for all tables"
  if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - send me the error." -ForegroundColor Red; return }
  Write-Host "MR created - merge it into development, and MR !17 picks it up." -ForegroundColor Green
}
git switch --quiet $orig
git branch -D $tmp 2>$null | Out-Null
Write-Host "Back on $(git rev-parse --abbrev-ref HEAD)." -ForegroundColor Green
}
