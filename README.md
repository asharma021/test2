uv run python -c "import yaml; c=yaml.safe_load(open('.gitlab-ci.yml', encoding='utf-8')); print('YAML syntax PASSED'); print('Postgres job exists:', 'postgres_extract_load' in c)"


git add .gitlab-ci.yml
git diff --cached --check
git diff --cached --stat
