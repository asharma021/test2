git add .gitlab-ci.yml src/extract_load/postgres/tables_iw_verify.yml
git diff --cached --check
git diff --cached --stat
git commit -m "Configure single-table PostgreSQL IVB verify-only job"


git push origin pipeline-testing
