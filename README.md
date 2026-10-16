# ============================================
# STEP 1 - CONFIRM YOUR FEATURE BRANCH
# ============================================

git branch --show-current
git status --short


# ============================================
# STEP 2 - FIND EL SHARED TEMPLATES
# ============================================

Select-String -Path .gitlab-ci.yml `
  -Pattern '^\.(el_upstream|el_job):' `
  -Context 0,100


# ============================================
# STEP 3 - FIND POSTGRESQL JOB
# ============================================

Select-String -Path .gitlab-ci.yml `
  -Pattern '^postgres_extract_load:' `
  -Context 0,15


# ============================================
# STEP 4 - FIND FAILURE HANDLING
# ============================================

Select-String -Path .gitlab-ci.yml `
  -Pattern 'allow_failure|exit |EL_MODULE|DBT_TARGET' `
  -Context 2,3


# ============================================
# STEP 5 - CHECK AVAILABLE EL COMMANDS
# ============================================

Get-Content src/extract_load/README.md -TotalCount 130
