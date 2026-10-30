
# Check current branch
git branch --show-current
git status --short

# Find full-refresh implementation
Get-ChildItem src/extract_load -Recurse -Filter *.py |
Select-String -Pattern 'full.refresh','full_refresh','load_mode','incremental' |
Select-Object -First 50 Path,LineNumber,Line

# Inspect known table mapping
Select-String -Path src/extract_load/postgres/tables.yml `
-Pattern 'ref_identifier_type' -Context 5,6

# Inspect mirroring implementation
Get-ChildItem src/extract_load -Recurse -Filter *.py |
Select-String -Pattern 'mirror' |
Select-Object -First 30 Path,LineNumber,Line
