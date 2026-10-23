# Review current changes
git diff -- .gitlab-ci.yml

# Check original production configuration
git show origin/development:.gitlab-ci.yml |
  Select-String -Pattern 'postgres_extract_load:' -Context 0,25

# Check current PostgreSQL configuration
Select-String -Path .gitlab-ci.yml `
  -Pattern '^postgres_extract_load:' `
  -Context 0,25

# Check branch and working tree
git status --short --branch

# Verify whitespace
git diff --check
