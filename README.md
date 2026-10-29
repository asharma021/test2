git diff --check
git status --short



uv run python -c "import yaml; c=yaml.safe_load(open('.gitlab-ci.yml',encoding='utf-8')); print('DEV:',c['postgres_extract_load_dev_incremental']['extends']); print('IVV:',c['postgres_extract_load_ivv_incremental']['extends']); print('Guard:',len(c['.postgres_incremental_guard']['before_script']))"
