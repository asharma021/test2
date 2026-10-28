Get-ChildItem src/extract_load -Recurse -Filter *.py |
Select-String -Pattern 'def resolve_env|resolve_env\s*='
