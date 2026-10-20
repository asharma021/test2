git branch --show-current
git diff --check
git diff -- .gitlab-ci.yml


git add .gitlab-ci.yml
git commit -m "Configure protected PostgreSQL EL testing branch"
git push origin feature/postgres-el-cicd-testing
