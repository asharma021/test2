& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
if (git status --porcelain --untracked-files=no) { Write-Host "Uncommitted changes - nothing done." -ForegroundColor Red; return }
git fetch origin --prune 2>$null
$orig = (git rev-parse --abbrev-ref HEAD).Trim()
"===== 1. BEFORE (development) - the actual dbt error line ====="
git switch --quiet --detach origin/development
Push-Location dbt
$out = @(uv run --no-sync dbt compile --target evaluator --profiles-dir . --select "test_name:no_deprecated_abbreviations" --target-path target\ab_before2 2>&1 | ForEach-Object { "$_" })
Pop-Location
git switch --quiet $orig
$out | Select-String -Pattern 'Error|does not exist|Catalog|not found' | Select-Object -First 8 | ForEach-Object { $_.Line }
"===== 2. Other compile-time DB lookups in models/macros/tests (development) ====="
git grep -n -E 'run_query\(|get_column_values\(|get_columns_in_relation\(|get_relations_by_pattern\(|statement\(' origin/development -- dbt/models dbt/macros dbt/tests | Where-Object { $_ -notmatch 'no_deprecated_abbreviations' } | Select-Object -First 25
"Back on: $(git rev-parse --abbrev-ref HEAD)"
}
