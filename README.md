uv run --no-sync python -c "import yaml; p='src/extract_load/postgres/tables_ivv_verify.yml'; d=yaml.safe_load(open(p)); print(d)"


Select-String -Path ".gitlab-ci.yml" `
  -Pattern 'postgres_extract_load_test:|EL_CONFIG|--config|extract-load-postgres' `
  -Context 5,15
