Get-Content '\\tthqpcelerra01\ttb_common\OA_Data_Sharing\OA\dbt_dev_env\extract_load.env' |
  Where-Object { $_ -match '^\s*[A-Za-z_][A-Za-z0-9_]*\s*=' } |
  ForEach-Object { ($_ -split '=', 2)[0].Trim() }
