Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>&1 | Out-Null
'--- Files in src/extract_load/postgres on origin/pipeline-testing ---'
git ls-tree --name-only origin/pipeline-testing src/extract_load/postgres/
'--- History of tables_ivv_verify.yml (A=added, D=deleted, M=modified) ---'
git log --oneline --all --name-status -- src/extract_load/postgres/tables_ivv_verify.yml
'--- Is it git-ignored? (no output = not ignored) ---'
git check-ignore -v src/extract_load/postgres/tables_ivv_verify.yml
'--- Local folder ---'
Get-ChildItem src\extract_load\postgres | Select-Object Name, Length, LastWriteTime
