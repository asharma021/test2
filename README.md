Select-String -Path src/extract_load/common/config.py `
  -Pattern 'source_profile|source=|source_group|table_groups' `
  -Context 4,6
