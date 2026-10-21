Get-Content .\src\extract_load\common\env.py |
Select-Object -Skip 55 -First 45


$path = $env:EL_ENV_FILE

if ([string]::IsNullOrWhiteSpace($path)) {
    Write-Output "EL_ENV_FILE is not configured"
}
elseif (-not (Test-Path -LiteralPath $path -PathType Leaf)) {
    Write-Output "EL environment file is not accessible"
}
else {
    Write-Output "EL environment file is accessible"
}
