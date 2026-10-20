Get-Content .\extract_load.env.example |
Select-String -Pattern "SRC_PG_|TARGET_PG_|DBT_PG_|EL_ENV_FILE" -Context 2,1
