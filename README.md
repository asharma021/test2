& {
$repo = 'C:\Users\SharmaAashishR\oa-business-layer'
Set-Location $repo
$tmp = 'tmp/sync-production'
$orig = (git rev-parse --abbrev-ref HEAD).Trim()
function Undo([string]$msg) {
  Write-Host "FAILED: $msg" -ForegroundColor Red
  git merge --abort 2>$null
  git reset --hard --quiet 2>$null
  git switch --quiet $orig 2>$null
  git branch -D $tmp 2>$null | Out-Null
  Write-Host "Reverted: back on '$(git rev-parse --abbrev-ref HEAD)'. Nothing pushed." -ForegroundColor Yellow
}
if (git status --porcelain --untracked-files=no) { Write-Host "Uncommitted changes - nothing done." -ForegroundColor Red; return }
git fetch origin --prune 2>$null
git branch -D $tmp 2>$null | Out-Null
"Commits on production not in development:"
git log --format='   %h %an : %s' origin/development..origin/production
git switch --quiet -c $tmp origin/development
if ($LASTEXITCODE -ne 0) { Write-Host "Could not create temp branch - nothing done." -ForegroundColor Red; return }
git merge --no-ff --no-commit origin/production 2>$null | Out-Null
$conf = @(git diff --name-only --diff-filter=U)
Write-Host "Conflicted files: $($conf -join ', ')" -ForegroundColor Cyan
if ($conf.Count -gt 0 -and (($conf -join '|') -ne 'uv.lock')) { Undo "conflicts other than uv.lock - send me this output"; return }
if ($conf.Count -eq 1) { git checkout --ours -- uv.lock; git add uv.lock }
git diff --cached --quiet origin/development
if ($LASTEXITCODE -ne 0) { git diff --cached --stat origin/development; Undo "merge result is not identical to development"; return }
Write-Host "OK: merge result is identical to development (history only)" -ForegroundColor Green
$msgFile = Join-Path $env:TEMP 'oa_sync_msg.txt'
$msg = "Merge production into development (keep development's uv.lock)`n`nproduction had one commit (uv.lock) not in development, which made the`npromotion to production conflict. This records it while keeping development's`nuv.lock (validated in DEV/IVV). No file content changes.`n"
[IO.File]::WriteAllText($msgFile, $msg, (New-Object Text.UTF8Encoding($false)))
git commit --quiet -F $msgFile
if ($LASTEXITCODE -ne 0) { Undo "commit failed (hook?) - see output above"; return }
git merge-tree --write-tree --name-only origin/production HEAD 2>&1 | Out-Null
if ($LASTEXITCODE -ne 0) { Undo "would still conflict with production"; return }
Write-Host "OK: development will now merge into production with no conflicts" -ForegroundColor Green
Write-Host "Pushing directly to development ..." -ForegroundColor Cyan
git push origin HEAD:development
if ($LASTEXITCODE -eq 0) {
  Write-Host "DONE: pushed to development. MR !17 now includes it." -ForegroundColor Green
} else {
  Write-Host "Direct push to development was rejected - opening a small MR instead." -ForegroundColor Yellow
  $br = 'chore/sync-production-into-development'
  git branch -m $tmp $br
  git push -u origin $br -o ci.skip -o merge_request.create -o merge_request.target=development -o "merge_request.title=Merge production into development (resolve uv.lock for promotion)"
  if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - send me the error." -ForegroundColor Red; return }
  Write-Host "MR created - merge it into development, and MR !17 picks it up." -ForegroundColor Green
  $tmp = $br
}
git switch --quiet $orig
git branch -D $tmp 2>$null | Out-Null
Write-Host "Back on $(git rev-parse --abbrev-ref HEAD)." -ForegroundColor Green
}
