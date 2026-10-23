# Show original inherited rules
git show origin/development:.gitlab-ci.yml |
  Select-String -Pattern '^\.el_upstream:' -Context 0,35

# Show current PostgreSQL job
Select-String -Path .gitlab-ci.yml `
  -Pattern '^postgres_extract_load:' -Context 0,25

# Check remaining changes
git status --short
git diff --check
