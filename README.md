git fetch origin
git switch pipeline-testing
git pull --ff-only origin pipeline-testing
git merge --ff-only feature/postgres-el-cicd-testing

git push origin pipeline-testing
