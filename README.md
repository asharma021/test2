& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
if (git status --porcelain --untracked-files=no) { Write-Host "Uncommitted changes - nothing done." -ForegroundColor Red; return }
git fetch origin --prune 2>$null
$orig = (git rev-parse --abbrev-ref HEAD).Trim()
$results = [ordered]@{}
foreach ($case in @(@{ Name = 'BEFORE'; Ref = 'origin/development' }, @{ Name = 'AFTER'; Ref = 'origin/fix/abbrev-test-compile' })) {
  Write-Host "`n===== $($case.Name): $($case.Ref) =====" -ForegroundColor Cyan
  git switch --quiet --detach $case.Ref
  if ($LASTEXITCODE -ne 0) { Write-Host "Could not check out $($case.Ref)" -ForegroundColor Red; $results[$case.Name] = 'NOT RUN'; continue }
  Push-Location dbt
  $out = @(uv run --no-sync dbt compile --target evaluator --profiles-dir . --select "test_name:no_deprecated_abbreviations" --target-path "target\ab_$($case.Name.ToLower())" 2>&1 | ForEach-Object { "$_" })
  $code = $LASTEXITCODE
  Pop-Location
  $out | Select-Object -Last 8
  $results[$case.Name] = if ($code -eq 0) { 'PASS' } else { 'FAIL' }
}
git switch --quiet $orig
Write-Host "`n===== RESULT =====" -ForegroundColor Cyan
$results.GetEnumerator() | ForEach-Object { "{0,-7} {1}" -f $_.Key, $_.Value }
if ($results['BEFORE'] -eq 'FAIL' -and $results['AFTER'] -eq 'PASS') { Write-Host "PROVEN: development fails on the missing seed; MR !9 compiles cleanly." -ForegroundColor Green }
else { Write-Host "Inconclusive - send me the output above." -ForegroundColor Yellow }
Write-Host "Back on: $(git rev-parse --abbrev-ref HEAD)" -ForegroundColor Green
git status --short --untracked-files=no
}
