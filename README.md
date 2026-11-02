Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>&1 | Out-Null
"===== Default branch ====="
(git remote show origin | Select-String 'HEAD branch').Line.Trim()
"===== Drift: commits only-in-production / only-in-development ====="
git rev-list --left-right --count origin/production...origin/development
"production is ancestor of development: $(git merge-base --is-ancestor origin/production origin/development; $LASTEXITCODE -eq 0)"
(git diff --stat origin/production origin/development | Select-Object -Last 1)
"===== production profiles.yml (structure only, no secrets) ====="
git show origin/production:dbt/profiles.yml | Select-String -Pattern '^\s*(target|host|dbname|schema|type):|^\s{2,4}\w+:\s*(&\w+)?\s*$' | Where-Object { $_.Line -notmatch '(?i)password|secret|token' } | ForEach-Object { "$($_.LineNumber): $($_.Line.TrimEnd())" }
"===== production .gitlab-ci.yml: how target/env are set ====="
git show origin/production:.gitlab-ci.yml | Select-String -Pattern 'DBT_TARGET|SKIP_EXTRACT_LOAD|EL_ENV_FILE|\.env' | Where-Object { $_.Line -notmatch '(?i)password|secret|token' } | ForEach-Object { "$($_.LineNumber): $($_.Line.Trim())" }
"===== production tables.yml non-default source groups ====="
git show origin/production:src/extract_load/postgres/tables.yml 2>$null | Select-String -Pattern '^\s*-\s*source:\s*\S+' -Context 0,2 | ForEach-Object { ($_.Line + ' ' + ($_.Context.PostContext -join ' ')).Trim() }
