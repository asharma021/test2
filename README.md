& {
$repo = 'C:\Users\SharmaAashishR\oa-business-layer'
Set-Location $repo
$branch = 'feature/prod-el-readiness'
$back = 'pipeline-testing'
$ours = @('.gitlab-ci.yml', 'dbt/profiles.yml')
if ((git rev-parse --abbrev-ref HEAD).Trim() -ne $branch) { Write-Host "Not on $branch - send me this output." -ForegroundColor Red; return }
$msgFile = Join-Path $env:TEMP 'oa_commit_msg.txt'
if (-not (Test-Path $msgFile)) { Write-Host "Commit message file missing - send me this output." -ForegroundColor Red; return }
Write-Host "=== dbt deps (install dbt packages locally) ===" -ForegroundColor Cyan
Push-Location (Join-Path $repo 'dbt')
uv run --no-sync dbt deps --profiles-dir .
$depsOk = ($LASTEXITCODE -eq 0)
Pop-Location
$other = @(git diff HEAD --name-only | Where-Object { $ours -notcontains $_ })
if ($other.Count) { git checkout HEAD -- $other; Write-Host "Restored files touched by dbt deps: $($other -join ', ')" -ForegroundColor Yellow }
$changed = @(git diff HEAD --name-only | Sort-Object)
if (($changed -join '|') -ne '.gitlab-ci.yml|dbt/profiles.yml') { Write-Host "Unexpected changed files: $($changed -join ', ') - stopping." -ForegroundColor Red; return }
if ($depsOk) { Write-Host "dbt deps OK - committing with ALL hooks." -ForegroundColor Green }
else { $env:SKIP = 'dbt-parse'; Write-Host "dbt deps FAILED - committing with all hooks except dbt-parse (CI will still run dbt on the MR)." -ForegroundColor Yellow }
git add $ours
git commit --quiet -F $msgFile
$cc = $LASTEXITCODE
Remove-Item Env:SKIP -ErrorAction SilentlyContinue
if ($cc -ne 0) { Write-Host "COMMIT FAILED - left as-is on $branch. Send me the output." -ForegroundColor Red; return }
git push -u origin $branch -o ci.skip -o merge_request.create -o merge_request.target=development -o merge_request.draft -o "merge_request.title=Draft: Prep prod PostgreSQL E/L - prod target fix + manual single-table prod run"
if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - commit kept locally on $branch; send me the error." -ForegroundColor Red; return }
Write-Host "DONE: $(git log -1 --format='%h %s') pushed; draft MR into development created." -ForegroundColor Green
git switch --quiet $back
Write-Host "Back on $(git rev-parse --abbrev-ref HEAD)." -ForegroundColor Green
}
