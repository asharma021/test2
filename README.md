& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
$ci = @(git show origin/development:.gitlab-ci.yml | ForEach-Object { $_.TrimEnd("`r") })
"===== 1. Top-level jobs/templates that look temporary ====="
for ($i = 0; $i -lt $ci.Count; $i++) {
  if ($ci[$i] -match '^([.\w-]+):\s*$' -and $ci[$i] -match '(dev_incremental|ivv_incremental|dev_verify|_test:|_inspect|incremental_guard|check_postgres|mirror_dev_verify)') { "{0,5}: {1}" -f ($i+1), $ci[$i] }
}
"===== 2. Every line mentioning pipeline-testing ====="
for ($i = 0; $i -lt $ci.Count; $i++) { if ($ci[$i] -match 'pipeline-testing') { "{0,5}: {1}" -f ($i+1), $ci[$i].Trim() } }
"===== 3. References to tables_ivv_verify.yml anywhere ====="
git grep -n 'tables_ivv_verify' origin/development -- . | Select-Object -First 20
"===== 4. Who uses .postgres_incremental_guard / EL_CONFIG ====="
for ($i = 0; $i -lt $ci.Count; $i++) { if ($ci[$i] -match 'postgres_incremental_guard|EL_CONFIG') { "{0,5}: {1}" -f ($i+1), $ci[$i].Trim() } }
}
