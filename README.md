Select-String -Path .gitlab-ci.yml -Pattern '^(check_el_env_loader|check_el_file_access|check_postgres_test_config|postgres_extract_load):' -Context 0,3
