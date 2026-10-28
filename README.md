Get-ChildItem "src/extract_load/common" -Recurse -Filter "*.py" |
Select-String -Pattern 'class TableMapping|source=|source_profile|group.get|DBT_TARGET' -Context 2,4
