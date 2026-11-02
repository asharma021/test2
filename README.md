& {
$repo = 'C:\Users\SharmaAashishR\oa-business-layer'
Set-Location $repo
$branch = 'feature/prod-el-readiness'
$back = 'pipeline-testing'
$cur = (git rev-parse --abbrev-ref HEAD).Trim()
Write-Host "Current branch: $cur" -ForegroundColor Cyan
if ($cur -ne $branch) { Write-Host "Not on $branch - send me this output." -ForegroundColor Red; return }
git fetch origin --prune 2>$null
$extra = (git rev-list --count origin/development..HEAD).Trim()
if ($extra -ne '0') { Write-Host "Branch has $extra commit(s) not in origin/development - stopping:" -ForegroundColor Red; git log --oneline origin/development..HEAD; return }
if (git ls-remote --heads origin $branch) { Write-Host "Remote $branch already exists - stopping." -ForegroundColor Red; return }
$changed = @(git diff HEAD --name-only | Sort-Object)
Write-Host "Changed files vs HEAD: $($changed -join ', ')" -ForegroundColor Cyan
if (($changed -join '|') -ne '.gitlab-ci.yml|dbt/profiles.yml') { Write-Host "Unexpected changed files - stopping (nothing touched)." -ForegroundColor Red; return }
$py = Join-Path $env:TEMP 'oa_validate_prod.py'
$msgFile = Join-Path $env:TEMP 'oa_commit_msg.txt'
if (-not (Test-Path $py) -or -not (Test-Path $msgFile)) { Write-Host "Helper files missing in TEMP - send me this output." -ForegroundColor Red; return }
uv run --no-sync python $py (Join-Path $repo '.gitlab-ci.yml') (Join-Path $repo 'dbt\profiles.yml')
if ($LASTEXITCODE -ne 0) { Write-Host "Validation FAILED - stopping (nothing committed)." -ForegroundColor Red; return }
Write-Host "`n===== DIFF =====" -ForegroundColor Cyan
git --no-pager diff HEAD
git add dbt/profiles.yml .gitlab-ci.yml
git commit --quiet -F $msgFile
if ($LASTEXITCODE -ne 0) { Write-Host "COMMIT FAILED (hook?) - left as-is on $branch. Send me the output above." -ForegroundColor Red; return }
git push -u origin $branch -o ci.skip -o merge_request.create -o merge_request.target=development -o merge_request.draft -o "merge_request.title=Draft: Prep prod PostgreSQL E/L - prod target fix + manual single-table prod run"
if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - commit kept locally on $branch; send me the error." -ForegroundColor Red; return }
Write-Host "`nDONE: $(git log -1 --format='%h %s') pushed; draft MR into development created." -ForegroundColor Green
git switch --quiet $back
Write-Host "Back on '$(git rev-parse --abbrev-ref HEAD)'." -ForegroundColor Green
}
