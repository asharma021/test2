Get-Content src/extract_load/common/env.py |
  Select-Object -Skip 75 -First 65

  Select-String -Path dbt/profiles.yml `
  -Pattern 'dev:|test:|prod:|env_var|target:' `
  -Context 2,5
