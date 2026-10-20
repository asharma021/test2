Get-ChildItem Env: |
Where-Object {
    $_.Name -match '^(SRC_PG_|EL_ENV_FILE)'
} |
Select-Object -ExpandProperty Name
