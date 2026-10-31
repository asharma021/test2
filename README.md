Set-Location C:\Users\SharmaAashishR\oa-business-layer
git add -- .gitlab-ci.yml
git commit -m "Configure DEV incremental test table and add DEV verify-only job" -m "postgres_extract_load_dev_incremental: add EL_CONFIG (tables.yml), EL_ENV_FILE (approved DEV env file), DBT_SELECT myttb_controlled_group; job remains disabled (when: never). New postgres_extract_load_dev_verify: manual, verify-only, same table/mapping/DEV target, pipeline-testing web pipelines only."
git push -o ci.skip origin pipeline-testing
git log --oneline -2
git status --short
