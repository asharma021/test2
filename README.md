Set-Location C:\Users\SharmaAashishR\oa-business-layer
'--- Env-file paths committed in HEAD ---'
git show HEAD:.gitlab-ci.yml | Select-String 'Data_Sharing' | ForEach-Object { "{0,5}: underscore={1}  {2}" -f $_.LineNumber, ($_.Line -match '\\_OA_Data_Sharing'), $_.Line.Trim() }
'--- Same check on origin/development ---'
git show origin/development:.gitlab-ci.yml | Select-String 'Data_Sharing' | ForEach-Object { "{0,5}: underscore={1}  {2}" -f $_.LineNumber, ($_.Line -match '\\_OA_Data_Sharing'), $_.Line.Trim() }
'--- Commits that changed the Data_Sharing path ---'
git log --oneline -G 'Data_Sharing' origin/development origin/pipeline-testing -- .gitlab-ci.yml | Select-Object -First 15
'--- Local check: which folder variant exists from this machine ---'
foreach ($d in '\\thqpcelerra01\ttb_common\OA_Data_Sharing\OA\dbt_dev_env', '\\thqpcelerra01\ttb_common\_OA_Data_Sharing\OA\dbt_dev_env') { "{0}  ->  {1}" -f $d, (Test-Path -LiteralPath $d) }
