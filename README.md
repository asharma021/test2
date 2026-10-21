Get-ChildItem .\src\extract_load -Recurse -Filter *.py |
Select-String -Pattern 'EL_ENV_FILE|load_dotenv|dotenv_values' |
Select-Object Path,LineNumber,Line
