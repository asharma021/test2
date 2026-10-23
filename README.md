git add .gitlab-ci.yml
git diff --cached --check
git diff --cached --stat
git commit -m "Configure PostgreSQL EL verify-only pipeline for development"
git push origin pipeline-testing
