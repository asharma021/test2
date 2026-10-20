Get-Content .\dbt\profiles.yml |
Select-String -Pattern 'dev:|host:|port:|dbname:|database:|schema:|user:' -Context 2,2
