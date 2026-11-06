# ---- 1. Env file: which source settings exist (names only) ----
$envFile = '\\thqpcelerra01\ttb_common\_OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'
Write-Host "`n=== Env file check ===" -ForegroundColor Cyan
if (-not (Test-Path $envFile)) { Write-Host "Env file not reachable" -ForegroundColor Red }
else {
    $found = @()
    Get-Content $envFile | ForEach-Object {
        if ($_ -match '^\s*(?:export\s+)?([A-Za-z0-9_]+)\s*=\s*(.*)$') {
            $name = $Matches[1]; $val = $Matches[2].Trim()
            if ($name -match '^(INFORMIX|ORACLE|ORA|MSSQL|SQLSERVER|POSTGRES|PG|MIRROR_PG)_') {
                $found += [pscustomobject]@{ Name = $name; Status = $(if ($val) {'SET'} else {'EMPTY'}) }
            }
        }
    }
    if ($found.Count -eq 0) { Write-Host "No source settings found in env file" -ForegroundColor Yellow }
    else { $found | Sort-Object Name | Format-Table -AutoSize }
}

# ---- 2. Pipeline config on production: how each E/L job gets its settings ----
Write-Host "`n=== production .gitlab-ci.yml (E/L lines) ===" -ForegroundColor Cyan
git fetch origin --quiet
$ci = git show origin/production:.gitlab-ci.yml
for ($i = 0; $i -lt $ci.Count; $i++) {
    if ($ci[$i] -match 'extract_load|EL_ENV_FILE|EL_MODULE|^\.el|extends:|prod_mirror') {
        "{0,5}: {1}" -f ($i + 1), $ci[$i]
    }
}
