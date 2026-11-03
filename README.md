& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
"===== A. MR !5 profiles.yml diff (PHOTO please) ====="
git --no-pager diff origin/development...origin/pipeline-testing -- dbt/profiles.yml
git log --format='%h %ad %an : %s' --date=short origin/development..origin/pipeline-testing -- dbt/profiles.yml
$ci = @(git show origin/pipeline-testing:.gitlab-ci.yml | ForEach-Object { $_.TrimEnd("`r") })
"===== B. dbt_compile job ====="
$i = [array]::IndexOf($ci, 'dbt_compile:')
if ($i -ge 0) { $e = $i + 1; while ($e -lt $ci.Count -and ($ci[$e] -match '^\s' -or $ci[$e] -eq '')) { $e++ }; $ci[$i..($e-1)] } else { '(dbt_compile not found)' }
"===== C. Where DBT_ALLOW_FAILURE / dbt_allow_failure is used ====="
$ci | Select-String 'DBT_ALLOW_FAILURE|dbt_allow_failure' | ForEach-Object { "$($_.LineNumber): $($_.Line.Trim())" }
"===== D. dbt_allow_failure input definition ====="
$ci | Select-String -Pattern '^\s+dbt_allow_failure:' -Context 0,4 | ForEach-Object { $_.Line; $_.Context.PostContext }
}
