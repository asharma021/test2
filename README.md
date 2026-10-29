python -c "import yaml; yaml.safe_load(open('src/extract_load/postgres/tables.yml')); print('YAML valid')"


Select-String -Path src/extract_load/postgres/tables.yml -Pattern "`t"
