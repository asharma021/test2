git diff --check
git add .gitlab-ci.yml dbt/profiles.yml
git commit -m "Configure PostgreSQL IVV verify-only pipeline"


git branch --show-current


git push origin pipeline-testing
