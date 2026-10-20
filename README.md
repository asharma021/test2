Get-Content .\src\extract_load\postgres\connections.py |
Select-String -Pattern 'SRC_PG_HOST|SRC_PG_PORT|SRC_PG_DATABASE|SRC_PG_USER|SRC_PG_PASSWORD' -Context 2,3
