# 1. Confirm current branch
git branch --show-current

# 2. Check working tree
git status

# 3. Fetch latest remote branches
git fetch origin

# 4. Confirm development branch exists
git branch -r

# 5. Show commits not yet in development
git log --oneline origin/development..origin/pipeline-testing

# 6. Show files changed compared with development
git diff --stat origin/development...origin/pipeline-testing

# 7. Show changed file names
git diff --name-status origin/development...origin/pipeline-testing

# 8. Review GitLab CI/CD differences
git diff origin/development...origin/pipeline-testing -- .gitlab-ci.yml

# 9. Review Python changes
git diff origin/development...origin/pipeline-testing -- src/extract_load/common/cli.py

# 10. Confirm working tree remains unchanged
git status --short --branch
