& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
if (git status --porcelain --untracked-files=no) { Write-Host "Uncommitted changes - nothing done." -ForegroundColor Red; return }
git fetch origin --prune 2>$null
git switch --quiet pipeline-testing
if ($LASTEXITCODE -ne 0) { Write-Host "Could not switch to pipeline-testing - nothing done." -ForegroundColor Red; return }
$behind = (git rev-list --count HEAD..origin/pipeline-testing).Trim()
$ahead  = (git rev-list --count origin/pipeline-testing..HEAD).Trim()
if ($behind -ne '0' -or $ahead -ne '0') { Write-Host "Local pipeline-testing differs from origin (behind $behind / ahead $ahead) - stopping." -ForegroundColor Red; return }
"===== Commits coming from development ====="
git log --oneline origin/pipeline-testing..origin/development
"===== Conflict check (in memory only) ====="
$out = git merge-tree --write-tree --name-only origin/pipeline-testing origin/development 2>&1
if ($LASTEXITCODE -ne 0) { Write-Host "CONFLICTS - nothing changed:" -ForegroundColor Red; $out | Select-Object -Skip 1; return }
Write-Host "Clean - merging." -ForegroundColor Green
git merge --no-ff --no-edit origin/development
if ($LASTEXITCODE -ne 0) { git merge --abort 2>$null; Write-Host "Merge failed - aborted, nothing changed." -ForegroundColor Red; return }
"===== Fix present on pipeline-testing? ====="
if (Select-String -Path dbt\tests\generic\no_deprecated_abbreviations.sql -SimpleMatch "flags.WHICH == 'compile'" -Quiet) { Write-Host "YES - guard is in place" -ForegroundColor Green } else { Write-Host "NO - resetting" -ForegroundColor Red; git reset --hard --quiet origin/pipeline-testing; return }
"===== MR !5 diff vs development after merge (should be only MR !5's own files) ====="
git diff --stat origin/development HEAD
git push origin pipeline-testing -o ci.skip
if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - merge kept locally; send me the error." -ForegroundColor Red; return }
Write-Host "DONE: $(git log -1 --format='%h %s') pushed to pipeline-testing (ci.skip)." -ForegroundColor Green
}
