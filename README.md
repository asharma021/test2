Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>&1 | Out-Null
foreach ($br in 'production', 'development', 'pipeline-testing') {
  "===== origin/$br  (last commit: $(git log -1 --format='%h %ad %s' --date=short origin/$br)) ====="
  $pf = @(git show "origin/${br}:dbt/profiles.yml" | ForEach-Object { $_.TrimEnd("`r") })
  $i = ($pf | Select-String -Pattern '^\s+prod:' | Select-Object -First 1).LineNumber
  '-- profiles.yml prod output (no secrets) --'
  if ($i) { $pf[($i - 1)..([Math]::Min($i + 14, $pf.Count - 1))] | Where-Object { $_ -notmatch '(?i)password|pass|secret|token' } } else { '(no prod output)' }
  $ci = @(git show "origin/${br}:.gitlab-ci.yml" | ForEach-Object { $_.TrimEnd("`r") })
  "-- .gitlab-ci.yml has spec:inputs: $([bool]($ci | Select-String -Pattern '^spec:')) --"
  $s = [array]::IndexOf($ci, 'postgres_extract_load:')
  if ($s -ge 0) { $e = $s + 1; while ($e -lt $ci.Count -and ($ci[$e] -match '^\s' -or $ci[$e] -eq '')) { $e++ }; $ci[$s..($e - 1)] } else { '(postgres_extract_load not found)' }
}
'===== tables.yml groups with a non-default source (would also run in prod) ====='
git show origin/pipeline-testing:src/extract_load/postgres/tables.yml | Select-String -Pattern '^\s*-?\s*source:\s*\S' -Context 0,2 |
  ForEach-Object { ($_.Line.Trim() + ' | ' + ($_.Context.PostContext -join ' ')).Trim() }
