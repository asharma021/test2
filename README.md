
Write-Host "=== PROJECT PYTHON REQUIREMENT ==="
Select-String -Path pyproject.toml -Pattern 'requires-python'

Write-Host "=== AVAILABLE PYTHON VERSIONS ==="
uv python list --only-installed

Write-Host "=== CURRENT ENVIRONMENT ==="
python --version
uv --version

Write-Host "=== GIT STATUS ==="
git status --short
