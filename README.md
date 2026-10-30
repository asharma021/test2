
# Inspect mirroring job configuration
Select-String -Path .gitlab-ci.yml `
  -Pattern '^prod_mirror:' `
  -Context 0,35

# Inspect mirror connection variable names only
Select-String -Path src/extract_load/mirror/connections.py `
  -Pattern 'MIRROR_PG_' `
  -Context 1,2

# Inspect current branch and changes
git branch --show-current
git status --short
