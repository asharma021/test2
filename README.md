$envFile = '\\tthqpcelerra01\ttb_common\_OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'

Get-Content -LiteralPath $envFile | ForEach-Object {
    if ($_ -match '^\s*([^#=\s]+)\s*=') {
        $key = $matches[1]
        if ($key -match '(?i)USER|PASS|PWD|SECRET|TOKEN|KEY|CREDENTIAL|AUTH|CONNECTION|DSN|URL|URI|HOST|SERVER|DATABASE|DB_NAME|PORT') {
            "$key=***HIDDEN***"
        } else {
            $_
        }
    } elseif ($_ -match '^\s*(#.*)?$') {
        $_
    } else {
        '[NON-STANDARD LINE HIDDEN]'
    }
}
