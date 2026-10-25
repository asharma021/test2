git diff --check
git diff -- .gitlab-ci.yml

git add .gitlab-ci.yml
git commit -m "Add PostgreSQL TEST environment access check"
git push -o ci.skip origin pipeline-testing
