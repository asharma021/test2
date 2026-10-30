
$envFile = '\\thqpcelerra01\ttb_common\_OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'

$required = @(
    'MIRROR_PG_HOST',
    'MIRROR_PG_DATABASE',
    'MIRROR_PG_USER',
    'MIRROR_PG_PASSWORD'
)

if (-not (Test-Path -LiteralPath $envFile -PathType Leaf)) {
    Write-Host 'DEV environment file not accessible'
} else {
    $names = Get-Content -LiteralPath $envFile |
        Where-Object { $_ -match '^\s*(MIRROR_PG_[A-Z_]+)\s*=' } |
        ForEach-Object { $Matches[1] }

    foreach ($key in $required) {
        if ($names -contains $key) {
            Write-Host "$key : PRESENT"
        } else {
            Write-Host "$key : MISSING"
        }
    }
}
