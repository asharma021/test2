# 1. Confirm the correct branch
git branch --show-current

# 2. Review staged files
git diff --cached --stat

# 3. Check staged changes for whitespace errors
git diff --cached --check

# 4. Commit the changes
git commit -m "Configure PostgreSQL DEV Extract Load verify-only pipeline"

# 5. Push to GitLab
git push origin pipeline-testing

# 6. Confirm local and remote status
git status

# 7. Display the latest commit
git log -1 --oneline
