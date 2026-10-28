Select-String -Path "src/extract_load/common/cli.py" `
  -Pattern "ConnectionPool|pool.source|source_profile" `
  -Context 4,6
