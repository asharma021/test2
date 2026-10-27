Select-String -Path src/extract_load/common/config.py `
  -Pattern '^def resolve_env' `
  -Context 0,45
