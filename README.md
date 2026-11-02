Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>&1 | Out-Null
"===== Commit(s) only on production ====="
git log origin/development..origin/production --format='%h %ad %an : %s' --date=short
"===== Files changed by that commit ====="
git log origin/development..origin/production --name-status --format=''
"===== Simulated merge development -> production (conflicted files, if any) ====="
git version
$out = git merge-tree --write-tree --name-only origin/production origin/development 2>&1
if ($LASTEXITCODE -eq 0) { "NO CONFLICTS - merge would be clean" } elseif ($LASTEXITCODE -eq 1) { "CONFLICTS:"; $out | Select-Object -Skip 1 | Where-Object { $_ -and $_ -notmatch '^(Auto-merging|CONFLICT)' } } else { "merge-tree not supported or failed:"; $out | Select-Object -First 5 }
"===== Extract/load and profiles files present on production ====="
git ls-tree -r --name-only origin/production | Select-String 'extract_load|profiles' | ForEach-Object { $_.Line }
