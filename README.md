Get-Content .\src\extract_load\env.example |
Select-String -Pattern 'SRC_PG_|TARGET_PG_|DBT_PG_' -Context 2,2
