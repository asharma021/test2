Select-String -Path .\src\extract_load\common\config.py `
  -Pattern 'EL_ENV_FILE|load_dotenv|dotenv|def resolve_env' `
  -Context 12,12
