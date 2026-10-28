Get-ChildItem "src/extract_load" -Recurse -Filter "*.py" |
Select-String -Pattern 'get_connection\(|pool.source\(|profile='
