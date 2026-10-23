# Confirm branch
git branch --show-current

# Check changes
git status --short
git diff --check

# Verify default branch
git symbolic-ref refs/remotes/origin/HEAD

# Inspect final PostgreSQL configuration
Get-Content .gitlab-ci.yml |
  Select-Object -Skip 543 -First 40

# Check Python changes relative to development
git diff origin/development...HEAD -- src/extract_load/common/cli.py

# Check all files changed relative to development
git diff --name-status origin/development...HEAD
