$config = @{}

Get-Content -LiteralPath $path | ForEach-Object {
    if ($_ -match '^\s*(SRC_PG_HOST|SRC_PG_PORT)\s*=\s*(.*)$') {
        $config[$matches[1]] = $matches[2].Trim().Trim('"', "'")
    }
}

if (-not $config['SRC_PG_HOST'] -or -not $config['SRC_PG_PORT']) {
    Write-Output "Source host or port is missing"
} else {
    Test-NetConnection `
        -ComputerName $config['SRC_PG_HOST'] `
        -Port ([int]$config['SRC_PG_PORT']) `
        -InformationLevel Quiet
}
