uv run --no-sync python -c "import yaml; c=yaml.safe_load(open('.gitlab-ci.yml', encoding='utf-8')); print('YAML syntax PASSED'); print('PostgreSQL job exists:', 'postgres_extract_load' in c)"


.\.venv\Scripts\python.exe -c "import yaml; c=yaml.safe_load(open('.gitlab-ci.yml', encoding='utf-8')); print('YAML syntax PASSED'); print('PostgreSQL job exists:', 'postgres_extract_load' in c)"

git add .gitlab-ci.yml
git diff --cached --check
git diff --cached --stat
