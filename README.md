git status --short --branch
git fetch origin
git diff --check origin/development...HEAD
git diff --stat origin/development...HEAD
git log --oneline origin/development..HEAD
git diff origin/development...HEAD -- .gitlab-ci.yml src/extract_load/common/cli.py
