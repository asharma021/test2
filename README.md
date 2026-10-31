
git branch --show-current
git status --short
git log -1 --oneline
git diff --check

# Confirm PostgreSQL and mirror source files
git ls-files src/extract_load/postgres/
git ls-files src/extract_load/mirror/

# Inspect the current CI job locations
Select-String -Path .gitlab-ci.yml `
  -Pattern '^\.el_job:', `
           '^\.postgres_incremental_guard:', `
           '^postgres_extract_load:', `
           '^postgres_extract_load_test:', `
           '^postgres_extract_load_dev_incremental:', `
           '^postgres_extract_load_ivv_incremental:', `
           '^prod_mirror:' |
  Select-Object LineNumber,Line
