Get-Content .\src\extract_load\common\config.py |
Select-String -Pattern 'EL_ENV_FILE|load_dotenv|resolve_env|dotenv' -Context 5,8
