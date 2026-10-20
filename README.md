Get-Content .\extract_load.env.example |
Select-String -Pattern 'SRC_PG_|DBT_PG_|TARGET_PG_|EL_ENV_FILE' |
ForEach-Object {
  ($_.Line -split '=', 2)[0]
}
