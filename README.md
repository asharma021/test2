Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>&1 | Out-Null
"HEAD == origin: $((git rev-parse HEAD) -eq (git rev-parse origin/pipeline-testing))"
git log --oneline -8 origin/pipeline-testing
git status --short
'--- Jobs now on pipeline-testing ---'
git show origin/pipeline-testing:.gitlab-ci.yml | Select-String -Pattern '^(\.?postgres_\w+|prod_mirror\w*|check_postgres_\w+|mirror_\w+):' | ForEach-Object { $_.Line }
'--- Env-file paths (underscore check) ---'
git show origin/pipeline-testing:.gitlab-ci.yml | Select-String 'EL_ENV_FILE:' | ForEach-Object { "underscore={0}  {1}" -f ($_.Line -match '\\_OA_Data_Sharing'), $_.Line.Trim() }Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>&1 | Out-Null
"HEAD == origin: $((git rev-parse HEAD) -eq (git rev-parse origin/pipeline-testing))"
git log --oneline -8 origin/pipeline-testing
git status --short
'--- Jobs now on pipeline-testing ---'
git show origin/pipeline-testing:.gitlab-ci.yml | Select-String -Pattern '^(\.?postgres_\w+|prod_mirror\w*|check_postgres_\w+|mirror_\w+):' | ForEach-Object { $_.Line }
'--- Env-file paths (underscore check) ---'
git show origin/pipeline-testing:.gitlab-ci.yml | Select-String 'EL_ENV_FILE:' | ForEach-Object { "underscore={0}  {1}" -f ($_.Line -match '\\_OA_Data_Sharing'), $_.Line.Trim() }
