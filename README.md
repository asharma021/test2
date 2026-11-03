& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
"===== Commits on MR !6 branch (newest first) ====="
git log --format='%h %ad : %s' --date=short origin/development..origin/feature/prod-el-readiness
"===== Default-branch rules on MR !6 branch right now ====="
$ci = @(git show origin/feature/prod-el-readiness:.gitlab-ci.yml | ForEach-Object { $_.TrimEnd("`r") })
foreach ($tpl in '.el_upstream:', '.dbt_env:', 'postgres_extract_load:') {
  $s = [array]::IndexOf($ci, $tpl); $e = $s + 1
  while ($e -lt $ci.Count -and ($ci[$e] -match '^\s' -or $ci[$e] -eq '')) { $e++ }
  $r = @(($s+1)..($e-1) | Where-Object { $ci[$_] -match 'CI_COMMIT_BRANCH == \$CI_DEFAULT_BRANCH' })[0]
  "--- $tpl"; $ci[$r..([Math]::Min($r+4, $e-1))]
}
}
