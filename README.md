& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
$ci = @(git show origin/development:.gitlab-ci.yml | ForEach-Object { $_.TrimEnd("`r") })
"===== 1. dbt_evaluate job ====="
$i = [array]::IndexOf($ci, 'dbt_evaluate:')
if ($i -ge 0) { $e = $i + 1; while ($e -lt $ci.Count -and ($ci[$e] -match '^\s' -or $ci[$e] -eq '')) { $e++ }; $ci[$i..($e-1)] } else { '(dbt_evaluate not found)' }
"===== 2. Code that writes the evaluator report / exits on issues ====="
git grep -n -E 'gl-code-quality-evaluate|evaluator_results' origin/development -- . ':!*.md' | Select-Object -First 20
"===== 3. Recent changes to the CI file and scripts ====="
git log -5 --format='%h %ad %an : %s' --date=short origin/development -- .gitlab-ci.yml scripts src/bl
}
