
# ============================================================
# PHASE 2 - POSTGRESQL EL EXECUTION ANALYSIS
# READ ONLY - NO DATABASE CONNECTIONS
# ============================================================

Write-Host "`n=== 1. VERIFY-ONLY IMPLEMENTATION ===" -ForegroundColor Cyan

Get-ChildItem src/extract_load -Recurse -Filter *.py |
    Select-String -Pattern 'verify.only|verify_only|verify-counts|verify_output' |
    Select-Object Path, LineNumber, Line


Write-Host "`n=== 2. POSTGRESQL RUNNER ===" -ForegroundColor Cyan

Get-Content src/extract_load/postgres/run.py -TotalCount 180


Write-Host "`n=== 3. POSTGRESQL ENTRY POINT ===" -ForegroundColor Cyan

Get-Content src/extract_load/postgres/__main__.py -TotalCount 100


Write-Host "`n=== 4. DATABASE TARGET RESOLUTION ===" -ForegroundColor Cyan

Get-Content src/extract_load/common/target.py -TotalCount 160


Write-Host "`n=== 5. CONNECTION CONFIGURATION REFERENCES ===" -ForegroundColor Cyan

Get-ChildItem src/extract_load -Recurse -Filter *.py |
    Select-String -Pattern 'DBT_TARGET|DBT_PG_USER|TARGET_PG_|EL_ENV_FILE|SRC_PG_' |
    Select-Object Path, LineNumber, Line


Write-Host "`n=== 6. CI EXECUTION AND BRANCH RULES ===" -ForegroundColor Cyan

$ci = Get-Content .gitlab-ci.yml
$ci[485..548]


Write-Host "`n=== 7. FINAL GIT STATUS ===" -ForegroundColor Cyan

git status --short
