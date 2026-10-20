Select-String -Path .\src\extract_load\postgres\connections.py -Pattern 'SRC_PG_HOST|SRC_PG_DATABASE|sslmode|psycopg.connect' -Context 3,5
