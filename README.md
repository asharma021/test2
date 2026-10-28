Get-ChildItem -Path . -Recurse -File -Include *.py,*.yml,*.yaml,*.env,*.md |
Where-Object { $_.FullName -notmatch '\\(\.venv|\.git)\\' } |
Select-String -Pattern 'python_srvc_acct|SRC_PG_USER|svc_acct'
