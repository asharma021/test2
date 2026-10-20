Get-ChildItem .\src\extract_load -Recurse -Filter *.py |
Select-String -Pattern 'DBT_PG_HOST|DBT_PG_USER|DBT_PG_PASSWORD|TARGET_PG|PG_TARGET|psycopg.connect'
