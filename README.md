Select-String -Path src/extract_load/postgres/*.py `
  -Pattern 'source_profile|source=|get_connection|source_schema|table_groups' `
  -Context 3,4
