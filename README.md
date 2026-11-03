& {
$repo = 'C:\Users\SharmaAashishR\oa-business-layer'
Set-Location $repo
$branch = 'feature/prod-el-readiness'
$orig = (git rev-parse --abbrev-ref HEAD).Trim()
function Undo([string]$msg) {
  Write-Host "FAILED: $msg" -ForegroundColor Red
  git reset --hard --quiet "origin/$branch" 2>$null
  git switch --quiet $orig 2>$null
  Write-Host "Reverted: $branch reset to origin, back on '$(git rev-parse --abbrev-ref HEAD)'. Nothing pushed." -ForegroundColor Yellow
}
if (git status --porcelain --untracked-files=no) { Write-Host "Uncommitted changes - nothing done." -ForegroundColor Red; return }
git fetch origin --prune 2>$null
git switch --quiet $branch
if ($LASTEXITCODE -ne 0) { Write-Host "Could not switch to $branch - nothing done." -ForegroundColor Red; return }
git reset --hard --quiet "origin/$branch"
$cp = Join-Path $repo '.gitlab-ci.yml'
$c = [IO.File]::ReadAllText($cp)
$nl = if ($c.Contains("`r`n")) { "`r`n" } else { "`n" }
$lines = New-Object System.Collections.ArrayList
$null = $lines.AddRange(($c -split "\r?\n"))
$ruleRx = '^\s+- if: [''"]?\$CI_COMMIT_BRANCH == \$CI_DEFAULT_BRANCH[''"]?\s*$'
foreach ($tpl in @('.el_upstream:', '.dbt_env:')) {
  $starts = @(0..($lines.Count-1) | Where-Object { $lines[$_] -eq $tpl })
  if ($starts.Count -ne 1) { Undo "$tpl found $($starts.Count) times"; return }
  $s = $starts[0]; $e = $s + 1
  while ($e -lt $lines.Count -and ($lines[$e] -match '^\s' -or $lines[$e] -eq '')) { $e++ }
  $hits = @(($s+1)..($e-1) | Where-Object { $lines[$_] -match $ruleRx })
  if ($hits.Count -ne 1) { Write-Host "Block $tpl :" -ForegroundColor Yellow; $lines[$s..($e-1)]; Undo "default-branch rule in $tpl found $($hits.Count) times (format differs)"; return }
  $r = $hits[0]
  if ($lines[$r+1] -notmatch '^\s+variables:\s*$' -or $lines[$r+2] -notmatch '^\s+DBT_TARGET:\s*"?prod"?\s*$') { Write-Host "Block $tpl :" -ForegroundColor Yellow; $lines[$s..($e-1)]; Undo "unexpected lines after default-branch rule in $tpl"; return }
  if ($lines[$r+3] -match '^\s+when:') { Undo "$tpl already has a when: on that rule"; return }
  $wi = $lines[$r+1] -replace '^(\s*).*$', '$1'
  $lines.Insert($r+3, $wi + 'when: manual')
  Write-Host "Edited $tpl (line $($r+4))" -ForegroundColor Cyan
}
[IO.File]::WriteAllText($cp, ($lines -join $nl), (New-Object Text.UTF8Encoding($false)))

$py = Join-Path $env:TEMP 'oa_validate_manual.py'
@'
import sys, yaml
class L(yaml.SafeLoader): pass
L.add_constructor('!reference', lambda l, n: ['!reference'] + l.construct_sequence(n))
ci = list(yaml.load_all(open(sys.argv[1], encoding='utf-8'), Loader=L))[-1]
def rule(name):
    rs = [r for r in ci[name]['rules'] if isinstance(r, dict) and r.get('if') == '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH']
    assert len(rs) == 1, (name, rs); return rs[0]
for n in ('.el_upstream', '.dbt_env'):
    r = rule(n); assert r.get('when') == 'manual' and r['variables']['DBT_TARGET'] == 'prod', (n, r); print('OK manual on default branch:', n, r)
p = [r for r in ci['postgres_extract_load']['rules'] if 'CI_DEFAULT_BRANCH' in str(r.get('if',''))][0]
assert p.get('when') == 'manual', p; print('OK postgres_extract_load still manual:', p)
for b in ('test', 'development'):
    rs = [r for r in ci['.dbt_env']['rules'] if isinstance(r, dict) and r.get('if') == f'$CI_COMMIT_BRANCH == "{b}"']
    assert len(rs) == 1 and 'when' not in rs[0], (b, rs); print(f'OK .dbt_env {b} rule unchanged:', rs[0])
'@ | Set-Content -Path $py -Encoding UTF8
uv run --no-sync python $py $cp
if ($LASTEXITCODE -ne 0) { Undo "validation failed"; return }
$changed = @(git diff --name-only)
if (($changed -join '|') -ne '.gitlab-ci.yml') { Undo "unexpected files changed: $($changed -join ', ')"; return }
Write-Host "`n===== DIFF =====" -ForegroundColor Cyan
git --no-pager diff

$msgFile = Join-Path $env:TEMP 'oa_manual_msg.txt'
@'
Make prod-writing jobs manual on the default branch for the prod bring-up

The default-branch rules of .el_upstream and .dbt_env set DBT_TARGET=prod and
ran automatically, so the first production pipeline after promotion would
start every extract/load (Oracle, SQL Server, Informix) and every dbt job that
writes to prod (seed, run, test, freshness, docs, elementary, diagnostics) at
once. Both rules are now when: manual, matching postgres_extract_load, so prod
writes only happen when someone plays the job. development and test rules are
unchanged. Remove the two 'when: manual' lines to restore full automation once
prod is validated.
'@ | Set-Content -Path $msgFile -Encoding UTF8
git add .gitlab-ci.yml
git commit --quiet -F $msgFile
if ($LASTEXITCODE -ne 0) { Undo "commit failed (hook?) - see output above"; return }
git push origin $branch -o ci.skip
if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - commit kept locally on $branch; send me the error." -ForegroundColor Red; return }
Write-Host "DONE: $(git log -1 --format='%h %s') pushed to $branch (MR !6, ci.skip)." -ForegroundColor Green
git switch --quiet $orig
Write-Host "Back on $(git rev-parse --abbrev-ref HEAD)." -ForegroundColor Green
}
