
git branch --show-current
git status --short

# Existing PostgreSQL E&L jobs
Select-String -Path .gitlab-ci.yml `
  -Pattern '^postgres_extract_load:', '^postgres_extract_load_dev_incremental:', '^postgres_extract_load_ivv_incremental:' `
  -Context 0,22

# Find existing mirroring jobs
Select-String -Path .gitlab-ci.yml `
  -Pattern 'mirror','mirroring','postgres' `
  -Context 2,5

# Inspect load-mode configuration
Select-String -Path src/extract_load/postgres/tables.yml `
  -Pattern 'incremental:','strategy:','unique_key:','cursor_column:' `
  -Context 2,4
