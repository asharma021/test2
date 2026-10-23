# Confirm the working branch
git branch --show-current

# Review all changes, including untracked files
git status --short

# Check for whitespace errors
git diff --check

# Inspect the PostgreSQL job
Get-Content .gitlab-ci.yml |
    Select-Object -Skip 540 -First 35

# Inspect the upstream rules
Get-Content .gitlab-ci.yml |
    Select-Object -Skip 510 -First 25

# Confirm the Python change is already committed
git diff -- src/extract_load/common/cli.py

# Check the untracked backup
git status --short -- .gitlab-ci.yml.backup
