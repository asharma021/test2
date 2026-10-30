git branch --show-current
git status --short
git diff --check

git add .gitlab-ci.yml

git diff --cached --check
git diff --cached

git commit -m "Add PostgreSQL DEV and IVV incremental load safety checks"

git push -o ci.skip origin HEAD

git status --short
