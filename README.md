git show origin/development:.gitlab-ci.yml | Select-String -Pattern '^---\s*$'


uv run --no-sync python -c "import yaml; docs=list(yaml.safe_load_all(open('.gitlab-ci.yml', encoding='utf-8'))); print('YAML syntax PASSED'); print('Documents:', len(docs)); print('PostgreSQL job exists:', any(isinstance(d,dict) and 'postgres_extract_load' in d for d in docs))"
