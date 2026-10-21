Select-String -Path .gitlab-ci.yml -Pattern 'EL_ENV_FILE|dotenv|load_dotenv|--env-file' -Context 3,6
