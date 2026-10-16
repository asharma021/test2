# ============================================
# STEP 5 - INSPECT SHARED EL CI TEMPLATES
# ============================================

Write-Host "`n=== CURRENT BRANCH ===" -ForegroundColor Cyan
git branch --show-current
git status --short

Write-Host "`n=== EL TEMPLATE LOCATIONS ===" -ForegroundColor Cyan
Select-String -Path .gitlab-ci.yml `
  -Pattern '^\.(el_job|el_upstream):'

Write-Host "`n=== SHARED EL TEMPLATES ===" -ForegroundColor Cyan
Select-String -Path .gitlab-ci.yml `
  -Pattern '^\.(el_job|el_upstream):' `
  -Context 0,65

Write-Host "`n=== PIPELINE INPUTS ===" -ForegroundColor Cyan
Get-Content .gitlab-ci.yml -TotalCount 110

Write-Host "`n=== POSTGRESQL TABLE CONFIGURATION ===" -ForegroundColor Cyan
Get-Content src/extract_load/postgres/tables.yml -TotalCount 100

Write-Host "`n=== POSTGRESQL CLI ENTRY POINT ===" -ForegroundColor Cyan
Get-Content src/extract_load/postgres/__main__.py -TotalCount 100

Write-Host "`n=== POSTGRESQL RUNNER CODE ===" -ForegroundColor Cyan
Get-Content src/extract_load/postgres/run.py -TotalCount 130
