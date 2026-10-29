
Get-ChildItem src/extract_load/postgres -Filter "tables*.yml" |
  Select-Object Name

Get-Content src/extract_load/postgres/tables.yml

Get-Content src/extract_load/postgres/tables_ivv_verify.yml
