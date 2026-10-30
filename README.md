
# Inspect supported load modes
Select-String -Path src/extract_load/common/cli.py `
  -Pattern 'full','incremental','verify','load_mode' `
  -Context 3,5

# Inspect PostgreSQL mapping for the known table
Select-String -Path src/extract_load/postgres/tables.yml `
  -Pattern 'myttb_cg_ref_identifier_type' `
  -Context 5,8

# Find mirroring jobs
Select-String -Path .gitlab-ci.yml `
  -Pattern '^prod_mirror:', 'mirror', 'mirroring' `
  -Context 3,8
