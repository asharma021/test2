Select-String -Path .gitlab-ci.yml -Pattern '^[a-zA-Z_][a-zA-Z0-9_-]*:' |
Where-Object { $_.LineNumber -ge 1390 } |
Select-Object LineNumber, Line
