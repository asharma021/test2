$path = '\\tthqpcelerra01\ttb_common\_OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'

Get-Content -LiteralPath $path |
Where-Object { $_ -match '^\s*SRC_PG_USER(_[A-Z]+)?\s*=' }
