
Write-Host "`n=== PHASE 3: CONFIGURATION INSPECTION ===" -ForegroundColor Cyan

Write-Host "`n1. CURRENT BRANCH" -ForegroundColor Yellow
git branch --show-current
git status --short

Write-Host "`n2. ENVIRONMENT FILE RESOLUTION" -ForegroundColor Yellow
Select-String -Path src/extract_load/common/env.py `
  -Pattern 'EL_ENV_FILE|DBT_PG_USER|DBT_PG_PASSWORD|TARGET_PG_USER|TARGET_PG_PASSWORD|def target_credentials|def target_endpoint' `
  -Context 3,8

Write-Host "`n3. VERIFICATION IMPLEMENTATION" -ForegroundColor Yellow
Select-String -Path src/extract_load/common/cli.py `
  -Pattern 'if args.verify_only|verify_requested|run_verification|verify_exit|return' `
  -Context 3,8

Write-Host "`n4. SOURCE CONFIGURATION KEYS" -ForegroundColor Yellow
Select-String -Path src/extract_load/postgres/connections.py `
  -Pattern 'SRC_PG_HOST|SRC_PG_PORT|SRC_PG_DATABASE|SRC_PG_USER|SRC_PG_PASSWORD' `
  -Context 1,3

Write-Host "`n5. CI ENVIRONMENT RULES" -ForegroundColor Yellow
Select-String -Path .gitlab-ci.yml `
  -Pattern 'EL_ENV_FILE|EL_VERIFY_ONLY|postgres_extract_load:|\.el_upstream:|allow_failure:' `
  -Context 2,6

Write-Host "`n=== INSPECTION COMPLETE ===" -ForegroundColor Green
