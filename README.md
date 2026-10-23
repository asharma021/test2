# 1. Check which files changed
git status --short

# 2. Confirm temporary diagnostic jobs are gone
Select-String -Path .gitlab-ci.yml -Pattern '^(check_postgres_test_config|check_el_file_access|check_el_env_loader):'

# 3. Check the number of removed lines
git diff --stat -- .gitlab-ci.yml
