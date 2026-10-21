git grep -n -i -E "OA_Data_Sharing|dbt_dev_env|extract_load\.env|EL_ENV_FILE"


Get-ChildItem -Path . -Recurse -File -Include *.md,*.yml,*.yaml,*.py,*.toml |
Select-String -Pattern 'OA_Data_Sharing','extract_load.env','EL_ENV_FILE' |
Select-Object Path,LineNumber,Line
