
# ================================================
# STEP 1 - PREPARE POSTGRESQL EL TEST
# Read-only checks; no database execution
# ================================================

Write-Host "`n=== BRANCH ===" -ForegroundColor Cyan
git branch --show-current
git status --short

Write-Host "`n=== PYTHON / UV ===" -ForegroundColor Cyan
python --version
uv --version

Write-Host "`n=== POSTGRESQL CLI ===" -ForegroundColor Cyan
uv run --locked extract-load-postgres --help
if ($LASTEXITCODE -ne 0) {
    throw "PostgreSQL EL CLI failed"
}

Write-Host "`n=== CI JOB RULES ===" -ForegroundColor Cyan
Select-String -Path .gitlab-ci.yml `
  -Pattern '^\.(el_job|el_upstream):|^postgres_extract_load:' `
  -Context 0,28

Write-Host "`n=== POSTGRESQL TABLE GROUPS ===" -ForegroundColor Cyan
Select-String -Path src/extract_load/postgres/tables.yml `
  -Pattern '^\s{0,6}(name|source|target|tags):' |
  Select-Object -First 65

Write-Host "`n=== FINAL STATUS ===" -ForegroundColor Cyan
git status --short

