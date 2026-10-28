$path = "\\tthqpcelerra01\ttb_common\OA_Data_Sharing\OA\dbt_dev_env\extract_load.env"

Get-Content -LiteralPath $path |
Where-Object { $_ -match '^\s*SRC_PG_(HOST|PORT|DATABASE|USER|PASSWORD)_IVW\s*=' } |
ForEach-Object { ($_ -split '=',2)[0].Trim() }
