
# ============================================================
# POSTGRESQL EL - PHASE 1: OFFLINE VALIDATION
# No database connections or data modifications
# ============================================================

Write-Host "`n=== CURRENT BRANCH ===" -ForegroundColor Cyan
git branch --show-current
git status --short

Write-Host "`n=== PYTHON VERSION ===" -ForegroundColor Cyan
python --version

Write-Host "`n=== UV VERSION ===" -ForegroundColor Cyan
uv --version

Write-Host "`n=== PYTHON SYNTAX VALIDATION ===" -ForegroundColor Cyan
python -m compileall -q src/extract_load

if ($LASTEXITCODE -ne 0) {
    Write-Host "Python syntax validation FAILED" -ForegroundColor Red
} else {
    Write-Host "Python syntax validation PASSED" -ForegroundColor Green
}

Write-Host "`n=== GIT WHITESPACE CHECK ===" -ForegroundColor Cyan
git diff --check

Write-Host "`n=== POSTGRESQL MODULE FILES ===" -ForegroundColor Cyan
Get-ChildItem src/extract_load/postgres -Filter *.py |
    Select-Object Name

Write-Host "`n=== TABLE CONFIGURATION ===" -ForegroundColor Cyan
Get-Item src/extract_load/postgres/tables.yml |
    Select-Object Name, Length

Write-Host "`n=== FINAL STATUS ===" -ForegroundColor Cyan
git status --short
