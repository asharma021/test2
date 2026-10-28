uv run --no-sync python -c "import yaml; p='src/extract_load/postgres/tables_ivv_verify.yml'; d=yaml.safe_load(open(p)); print(d)"


python -c "import yaml; p='src/extract_load/postgres/tables_ivv_verify.yml'; d=yaml.safe_load(open(p)); print(d)"
