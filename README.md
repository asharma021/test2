$envFile = '\\thqpcelerra01\ttb_common\_OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'
if (-not (Test-Path -LiteralPath $envFile)) { Write-Host "Env file not reachable" -ForegroundColor Red; return }
$found = @()
foreach ($line in [System.IO.File]::ReadAllLines($envFile)) {
    if ($line -match '^\s*(?:export\s+)?([A-Za-z0-9_]+)\s*=\s*(.*)$') {
        $name = $Matches[1]; $val = $Matches[2].Trim()
        if ($name -match '^(INFORMIX|ORACLE|ORA|MSSQL|SQLSERVER)_') {
            $found += [pscustomobject]@{ Name = $name; Status = $(if ($val) {'SET'} else {'EMPTY'}) }
        }
    }
}
if ($found.Count -eq 0) { Write-Host "NONE: no Oracle / SQL Server / Informix settings in env file" -ForegroundColor Yellow }
else { $found | Sort-Object Name | Format-Table -AutoSize }
