& {
$repo = 'C:\Users\SharmaAashishR\oa-business-layer'
Set-Location $repo
$branch = 'feature/prod-el-readiness'
$orig = (git rev-parse --abbrev-ref HEAD).Trim()
function Undo([string]$msg) {
  Write-Host "FAILED: $msg" -ForegroundColor Red
  git revert --abort 2>$null
  git reset --hard --quiet "origin/$branch" 2>$null
  git switch --quiet $orig 2>$null
  Write-Host "Reverted: $branch reset to origin, back on '$(git rev-parse --abbrev-ref HEAD)'. Nothing pushed." -ForegroundColor Yellow
}
if (git status --porcelain --untracked-files=no) { Write-Host "Uncommitted changes - nothing done." -ForegroundColor Red; return }
git fetch origin --prune 2>$null
$list = @(git log --format='%H|%h %s' "origin/development..origin/$branch")
"===== Commits on MR !6 branch ====="
$list | ForEach-Object { ($_ -split '\|', 2)[1] }
if ($list | Where-Object { $_ -match 'Revert manual gating' }) { Write-Host "Revert already on the branch - nothing to do." -ForegroundColor Yellow; return }
$hit = @($list | Where-Object { $_ -match 'Make prod-writing jobs manual' })
if ($hit.Count -ne 1) { Write-Host "Expected exactly 1 manual-gating commit, found $($hit.Count) - nothing done." -ForegroundColor Red; return }
$target = ($hit[0] -split '\|', 2)[0]
Write-Host "Will revert: $(($hit[0] -split '\|', 2)[1])" -ForegroundColor Cyan
git switch --quiet $branch
if ($LASTEXITCODE -ne 0) { Write-Host "Could not switch to $branch - nothing done." -ForegroundColor Red; return }
git reset --hard --quiet "origin/$branch"
$msgFile = Join-Path $env:TEMP 'oa_revert_msg.txt'
$msg = "Revert manual gating of non-Postgres prod jobs`n`nKeep only postgres_extract_load manual on the default branch. Oracle, SQL`nServer, Informix extract/load and the dbt jobs keep their original behaviour.`n"
[IO.File]::WriteAllText($msgFile, $msg, (New-Object Text.UTF8Encoding($false)))
git revert --no-edit --no-commit $target
if ($LASTEXITCODE -ne 0) { Undo "revert had conflicts"; return }
git commit --quiet -F $msgFile
if ($LASTEXITCODE -ne 0) { Undo "commit failed (hook?)"; return }
$py = Join-Path $env:TEMP 'oa_validate_revert.py'
@'
import sys, yaml
class L(yaml.SafeLoader): pass
L.add_constructor('!reference', lambda l, n: ['!reference'] + l.construct_sequence(n))
ci = list(yaml.load_all(open(sys.argv[1], encoding='utf-8'), Loader=L))[-1]
for n in ('.el_upstream', '.dbt_env'):
    r = [x for x in ci[n]['rules'] if isinstance(x, dict) and x.get('if') == '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'][0]
    assert 'when' not in r, (n, r); print('OK back to original (automatic):', n, r)
p = [x for x in ci['postgres_extract_load']['rules'] if 'CI_DEFAULT_BRANCH' in str(x.get('if',''))][0]
assert p.get('when') == 'manual', p; print('OK postgres_extract_load manual:', p)
'@ | Set-Content -Path $py -Encoding UTF8
uv run --no-sync python $py (Join-Path $repo '.gitlab-ci.yml')
if ($LASTEXITCODE -ne 0) { Undo "validation failed"; return }
Write-Host "`n===== MR !6 total change vs development (expect 2 files) =====" -ForegroundColor Cyan
git diff --stat origin/development...HEAD
git push origin $branch -o ci.skip
if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - commit kept locally on $branch; send me the error." -ForegroundColor Red; return }
Write-Host "DONE: $(git log -1 --format='%h %s') pushed (ci.skip)." -ForegroundColor Green
git switch --quiet $orig
Write-Host "Back on $(git rev-parse --abbrev-ref HEAD)." -ForegroundColor Green
}
