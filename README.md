
# Confirm current branch
git branch --show-current

# Fetch latest changes
git fetch origin

# Check working tree
git status --short

# Check whitespace
git diff --check

# Check Python syntax
.\.venv\Scripts\python.exe -m py_compile src/extract_load/common/cli.py

# Check files changed compared with development
git diff --name-status origin/development...HEAD

# Review uncommitted CI changes
git diff -- .gitlab-ci.yml

# Review committed Python changes
git diff origin/development...HEAD -- src/extract_load/common/cli.py

# Check for unexpected tracked changes
git diff --stat origin/development...HEAD

# Check branch divergence
git rev-list --left-right --count origin/development...HEAD

# Confirm current status
git status --short --branch
