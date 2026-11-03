& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
if (git status --porcelain --untracked-files=no) { Write-Host "Uncommitted changes - nothing done." -ForegroundColor Red; return }
git fetch origin --prune 2>$null
$orig = (git rev-parse --abbrev-ref HEAD).Trim()
git switch --quiet --detach origin/development
"===== 1. Exit points in the report script ====="
git grep -n -E 'sys\.exit|SystemExit|return 1|raise ' HEAD -- scripts/gitlab_code_quality.py | Select-Object -First 20
"===== 2. Evaluator build (offline DuckDB) ====="
Push-Location dbt
$b = @(uv run --no-sync dbt build --select "package:dbt_project_evaluator" dbt_project_evaluator_exceptions --target evaluator --profiles-dir . --vars '{print_evaluator_issues: true}' --log-path logs\eval_local 2>&1 | ForEach-Object { "$_" })
"dbt build exit code: $LASTEXITCODE"
$b | Select-Object -Last 3
Pop-Location
"===== 3. The report script (this is the step that fails in CI) ====="
$out = Join-Path $env:TEMP 'gl-eval-test.json'
uv run --no-sync python scripts/gitlab_code_quality.py --tool dbt-evaluator --duckdb dbt\evaluator.duckdb --manifest dbt\target\manifest.json --run-results dbt\target\run_results.json --output $out 2>&1 | ForEach-Object { "$_" }
"report script exit code: $LASTEXITCODE"
"report file created: $(Test-Path $out)"
git switch --quiet $orig
"Back on: $(git rev-parse --abbrev-ref HEAD)"
git status --short --untracked-files=no
}
