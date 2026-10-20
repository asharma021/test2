Get-Content .\src\extract_load\postgres\run.py |
Select-String -Pattern "verify.only|verify_only|get_connection|SRC_PG" -Context 4,5
