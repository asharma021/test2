Get-Content .\src\extract_load\common\cli.py |
Select-String -Pattern "verify.only|verify_only|get_connection|count_rows|connect_source" -Context 10,15
