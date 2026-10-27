if ($env:DBT_TARGET -eq "test" -and $env:EL_MODULE -eq "postgres") {
    Write-Host "Configuring IVV PostgreSQL source"

    if (-not $env:SRC_PG_HOST_IVV -or
        -not $env:SRC_PG_PORT_IVV -or
        -not $env:SRC_PG_DATABASE_IVV) {
        throw "Missing IVV PostgreSQL source configuration"
    }

    $env:SRC_PG_HOST = $env:SRC_PG_HOST_IVV
    $env:SRC_PG_PORT = $env:SRC_PG_PORT_IVV
    $env:SRC_PG_DATABASE = $env:SRC_PG_DATABASE_IVV

    Write-Host "IVV source configuration selected"
}
