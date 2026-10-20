Get-Content .\src\extract_load\postgres\run.py |
Select-String -Pattern "verify.only|verify_only|get_connection|count_rows|extract_batches" -Context 8,12
