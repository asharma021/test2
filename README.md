
# =====================================================
# PHASE 2 - POSTGRESQL EL EXECUTION ANALYSIS
# READ ONLY - NO DATABASE MODIFICATIONS
# =====================================================

Write-Host "`n=== VERIFY-ONLY IMPLEMENTATION ===" -ForegroundColor Cyan

Get-ChildItem src/extract_load -Recurse -Filter *.py |
    Select-String -Pattern 'verify.only|verify_only|verify-counts|verify_output' |
    Select-Object Path, LineNumber, Line


Write-Host "`n=== POSTGRESQL RUNNER ===" -ForegroundColor Cyan

Get-Content src/extract_load/postgres/run.py -TotalCount 180


Write-Host "`n=== POSTGRESQL ENTRY POINT ===" -ForegroundColor Cyan

Get-Content src/extract_load/postgres/__main__.py -TotalCount 100


Write-Host "`n=== DATABASE TARGET RESOLUTION ===" -ForegroundColor Cyan

Get-Content src/extract_load/common/target.py -TotalCount 160


Write-Host "`n=== POSTGRESQL CONNECTION HANDLING ===" -ForegroundColor Cyan

Get-Content src/extract_load/postgres/connections.py -TotalCount 140


Write-Host "`n=== FAILURE HANDLING ===" -ForegroundColor Cyan

Get-ChildItem src/extract_load/postgres -Filter *.py |
    Select-String -Pattern 'raise|rollback|commit|sys.exit|return False' |
    Select-Object Path, LineNumber, Line


Write-Host "`n=== FINAL STATUS ===" -ForegroundColor Cyan

git status --short
