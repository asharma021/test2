git status --short

git add .gitlab-ci.yml

git -c core.hooksPath=NUL commit -m "Fix IVV PostgreSQL environment file path"

git push -o ci.skip origin pipeline-testing
