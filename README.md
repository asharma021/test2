

# ============================================================
# POSTGRESQL EXTRACT/LOAD - DEVOPS DEV TESTING
# Project: OA Business Layer
# Base branch: development
# Working branch: feature/postgres-el-cicd-testing
# ============================================================


# ============================================================
# STEP 1 - OPEN YOUR ALREADY CLONED REPOSITORY
# ============================================================

# First navigate into your existing repository folder.
# Replace the example path with your actual local path.

# cd /c/Users/YOUR_USERNAME/oa-business-layer

pwd
git status
git remote -v

# Confirm you are in the correct GitLab repository.


# ============================================================
# STEP 2 - FETCH LATEST DEVELOPMENT CODE
# ============================================================

git fetch origin

# Check for local changes before switching branches.
git status --short

# Do not continue if you have uncommitted work
# that you need to preserve.

git switch development
git pull --ff-only origin development

# Confirm current branch and latest commit.

git branch --show-current
git log -1 --oneline


# ============================================================
# STEP 3 - CREATE YOUR OWN FEATURE BRANCH
# ============================================================

git switch -c feature/postgres-el-cicd-testing

# Verify branch.

git branch --show-current
git status

# Push your branch to GitLab.
# This can trigger an existing branch pipeline,
# so inspect .gitlab-ci.yml rules before pushing.

# git push -u origin feature/postgres-el-cicd-testing


# ============================================================
# STEP 4 - OPEN PROJECT IN VS CODE
# ============================================================

code .

# If 'code' is not recognized, open VS Code manually.


# ============================================================
# STEP 5 - INSPECT PROJECT STRUCTURE
# ============================================================

ls -la

ls src/extract_load

ls dbt

# Inspect the pipeline.
sed -n '1,120p' .gitlab-ci.yml

# Find PostgreSQL EL job.
grep -n -A 90 '^postgres_extract_load:' .gitlab-ci.yml

# Find runner tags.
grep -n -A 5 '^default:' .gitlab-ci.yml

# Find environment configuration.
grep -n -A 50 '^\.dbt_env:' .gitlab-ci.yml


# ============================================================
# STEP 6 - REVIEW TARGET ENVIRONMENT
# ============================================================

# Inspect profiles.yml.
# Do not print or share actual credentials.

grep -n -A 22 'dev:' dbt/profiles.yml

# Review EL example variable names.
grep -nE '^[A-Z][A-Z0-9_]*=' extract_load.env.example

# Review EL source configuration.
grep -nE 'postgres|incremental|cursor_column|unique_key|fuzz' \
  src/extract_load/tables.yml


# ============================================================
# STEP 7 - INSPECT PYTHON ENTRY POINT
# ============================================================

# Locate the actual PostgreSQL EL implementation.

find src/extract_load -type f -name '*.py'

# Find CLI arguments and configuration handling.

grep -R -nE \
  'argparse|--select|--full-refresh|DBT_TARGET|EL_ENV_FILE' \
  src/extract_load

# Find failure handling.

grep -R -nE \
  'sys.exit|SystemExit|raise|rollback|commit|watermark' \
  src/extract_load


# ============================================================
# STEP 8 - VERIFY AVAILABLE TOOLS
# ============================================================

git --version
python --version
uv --version

# Check whether Python project dependencies are available.
ls pyproject.toml uv.lock

# Do not install dependencies yet if the project uses
# internal package repositories or an approved setup process.


# ============================================================
# STEP 9 - OPTIONAL OFFLINE CODE VALIDATION
# ============================================================

# These commands should not run source extraction.
# Use only after dependencies are available.

# Check Python syntax.
python -m compileall -q src/extract_load

# Check Git changes.
git diff --check

# Check branch status.
git status


# ============================================================
# STEP 10 - PREPARE FOR CONTROLLED DEV PIPELINE TEST
# ============================================================

# Before executing PostgreSQL EL, confirm:
#
# 1. Latest job rules allow feature-branch testing.
# 2. DBT_TARGET resolves to dev.
# 3. DEV database host and schema are correct.
# 4. EL_ENV_FILE or approved variables are available.
# 5. Source credentials are valid.
# 6. Runner python-app05 is available.
# 7. DBT_SELECT syntax has been verified.
# 8. Selected tables are approved for DEV testing.
# 9. The operation cannot write to PROD.
#
# Do not run extraction until these checks pass.


# ============================================================
# STEP 11 - COMMIT CHANGES (AFTER MAKING/REVIEWING THEM)
# ============================================================

# Review changes first.
git diff
git diff --check

# Stage only the files you intentionally modified.
# Example:
# git add .gitlab-ci.yml

# Commit after reviewing.
# git commit -m "test: validate PostgreSQL EL in DEV"

# Push after checking the CI rules.
# git push -u origin feature/postgres-el-cicd-testing


# ============================================================
# STEP 12 - VERIFY
# ============================================================

git status
git branch --show-current
git log -3 --oneline

# Then check the feature-branch pipeline in GitLab.
#
# GitLab -> Build -> Pipelines
#
# Inspect:
# - Job inclusion/exclusion
# - Runner assignment
# - Target environment
# - Python exit code
# - EL logs and artifacts
# - Table-level success/failure
# ============================================================
