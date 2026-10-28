Get-ChildItem "src/extract_load" -Recurse -Filter "*.py" |
Select-String -Pattern "class ConnectionPool|def source\(" -Context 5,15
