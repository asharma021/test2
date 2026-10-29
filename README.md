
git diff --name-status origin/development...HEAD

git log --oneline --left-right origin/development...HEAD

git diff origin/development...HEAD -- .gitlab-ci.yml
