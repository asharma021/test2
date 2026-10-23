# Review the complete PostgreSQL job
Get-Content .gitlab-ci.yml |
Select-Object -Skip 540 -First 35

# Review upstream job configuration
Get-Content .gitlab-ci.yml |
Select-Object -Skip 300 -First 85

# Check exact YAML changes
git diff -- .gitlab-ci.yml

# Confirm only intended changes remain
git status --short

# Check whitespace
git diff --check
