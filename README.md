
# ==================================================
# STEP 2 - INSPECT UPDATED POSTGRESQL EL PIPELINE
# ==================================================

Select-String -Path .gitlab-ci.yml -Pattern '^postgres_extract_load:' -Context 0,90


# ==================================================
# STEP 3 - INSPECT EXTRACT/LOAD PYTHON FILES
# ==================================================

# List all Python files
Get-ChildItem src/extract_load -Recurse -Filter *.py |
Select-Object FullName

# Check PostgreSQL configuration directory
Get-ChildItem src/extract_load/postgres


# ==================================================
# STEP 4 - REVIEW EL DOCUMENTATION
# ==================================================

# Read Extract/Load README
Get-Content src/extract_load/README.md -TotalCount 100

# Read onboarding instructions
Get-Content docs/onboarding/12-new-data-end-to-end.md -TotalCount 100


# ==================================================
# VERIFY CURRENT BRANCH
# ==================================================

git branch --show-current
git status
