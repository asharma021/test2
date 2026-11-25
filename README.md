$f = '\\thqpcelerra01\ttb_common\_OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'
Select-String -Path $f -Pattern '^\s*SRC_PG_\w*\s*=' |
  ForEach-Object { ($_.Line -split '=', 2)[0].Trim() } | Sort-Object
