# 1. Confirm branch
git branch --show-current

# 2. Check changes
git status

# 3. Validate whitespace
git diff --check

# 4. Stage only the GitLab YAML file
git add .gitlab-ci.yml

# 5. Check staged changes
git diff --cached --stat
git diff --cached --check

# 6. Commit changes
git commit -m "Add PostgreSQL TEST environment diagnostic job"

# 7. Push to GitLab
git push origin pipeline-testing

# 8. Verify local and remote status
git status

# 9. Display latest commit
git log -1 --oneline
