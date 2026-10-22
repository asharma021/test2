$path = '\\thqpcelerra01\ttb_common\_OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'

Test-Path -LiteralPath $path -PathType Leaf


Get-Content -LiteralPath $path |
Where-Object {
    $_ -match '^\s*SRC_PG_(HOST|PORT|DATABASE|USER)\s*='
} |
ForEach-Object {
    ($_ -split '=', 2)[0]
}
