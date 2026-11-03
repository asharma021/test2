& {
$repo = 'C:\Users\SharmaAashishR\oa-business-layer'
Set-Location $repo
$branch = 'ci/env-setup-auto-on-development'
$orig = (git rev-parse --abbrev-ref HEAD).Trim()
function Undo([string]$msg) {
  Write-Host "FAILED: $msg" -ForegroundColor Red
  git reset --hard --quiet "origin/$branch" 2>$null
  git switch --quiet $orig 2>$null
  Write-Host "Reverted: back on '$(git rev-parse --abbrev-ref HEAD)'. Nothing pushed." -ForegroundColor Yellow
}
if (git status --porcelain --untracked-files=no) { Write-Host "Uncommitted changes - nothing done." -ForegroundColor Red; return }
git fetch origin --prune 2>$null
git switch --quiet $branch
if ($LASTEXITCODE -ne 0) { Write-Host "Could not switch to $branch - nothing done." -ForegroundColor Red; return }
git reset --hard --quiet "origin/$branch"
$cp = Join-Path $repo '.gitlab-ci.yml'
$py = Join-Path $env:TEMP 'oa_edit_envsetup_comment.py'
@'
import sys
p = sys.argv[1]
raw = open(p, 'rb').read().decode('utf-8')
bom = raw.startswith('\ufeff')
if bom: raw = raw[1:]
nl = '\r\n' if '\r\n' in raw else '\n'
lines = raw.split(nl)
hits = [i for i, l in enumerate(lines) if 'create the pipeline but wait for a play' in l]
if len(hits) != 1: sys.exit(f'comment line 1 found {len(hits)} times')
i = hits[0]
if 'one click releases them all' not in lines[i + 1]: sys.exit(f'unexpected next line: {lines[i+1]!r}')
print('Removing:', lines[i].strip()); print('Removing:', lines[i + 1].strip())
del lines[i:i + 2]
open(p, 'wb').write((('\ufeff' if bom else '') + nl.join(lines)).encode('utf-8'))
'@ | Set-Content -Path $py -Encoding UTF8
uv run --no-sync python $py $cp
if ($LASTEXITCODE -ne 0) { Undo "comment edit failed - see message above"; return }
$changed = @(git diff --name-only)
if (($changed -join '|') -ne '.gitlab-ci.yml') { Undo "unexpected files: $($changed -join ', ')"; return }
git --no-pager diff
$msgFile = Join-Path $env:TEMP 'oa_envsetup_comment_msg.txt'
[IO.File]::WriteAllText($msgFile, "Remove outdated comment about manual env_setup on development`n", (New-Object Text.UTF8Encoding($false)))
git add .gitlab-ci.yml
git commit --quiet -F $msgFile
if ($LASTEXITCODE -ne 0) { Undo "commit failed (hook?) - see output above"; return }
git push origin $branch -o ci.skip
if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - commit kept locally on $branch; send me the error." -ForegroundColor Red; return }
Write-Host "DONE: $(git log -1 --format='%h %s') pushed to MR !16 (ci.skip)." -ForegroundColor Green
git switch --quiet $orig
Write-Host "Back on $(git rev-parse --abbrev-ref HEAD)." -ForegroundColor Green
}
