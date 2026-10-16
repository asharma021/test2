
# PHASE 3 - SAFE CONFIGURATION CHECK
# No database connections or modifications

Write-Host "`n=== CURRENT BRANCH ===" -ForegroundColor Cyan
git branch --show-current

Write-Host "`n=== ENVIRONMENT RESOLUTION ===" -ForegroundColor Cyan
Select-String -Path src/extract_load/common/env.py `
  -Pattern 'def target_credentials|def target_endpoint|def resolve_env|EL_ENV_FILE' `
  -Context 2,15

Write-Host "`n=== CLI VERIFY BEHAVIOR ===" -ForegroundColor Cyan
Select-String -Path src/extract_load/common/cli.py `
  -Pattern 'verify_only|verify_counts|run_verification' `
  -Context 4,10

Write-Host "`n=== POSTGRESQL CONFIGURATION ===" -ForegroundColor Cyan
Get-Content src/extract_load/postgres/tables.yml -TotalCount 75

Write-Host "`n=== CI JOB RULES ===" -ForegroundColor Cyan
Select-String -Path .gitlab-ci.yml `
  -Pattern 'postgres_extract_load:|\.el_upstream:|allow_failure: true' `
  -Context 4,12

Write-Host "`n=== WORKING TREE ===" -ForegroundColor Cyan
git status --short
