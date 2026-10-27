Select-String -Path src/extract_load/postgres/connections.py `
  -Pattern 'SRC_PG_|load_el_env|os.getenv|os.environ' `
  -Context 5,5
