git diff --check
git status --short


git add .gitlab-ci.yml
git commit -m "Configure TEST environment access check on protected branch"
git push -u origin feature/postgres-el-test-validation
