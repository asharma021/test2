
# Confirm Python requirement
Select-String -Path pyproject.toml -Pattern 'requires-python'

# Check installed Python versions
uv python list --only-installed

# Check virtual environment
Test-Path .venv

# Check available disk space
Get-PSDrive C

# Check package cache location
uv cache dir

# Confirm Git status
git status --short
