Set-Location C:\Users\SharmaAashishR\oa-business-layer
"===== production: root profiles.yml targets (no secrets) ====="
git show origin/production:profiles.yml | Select-String -Pattern '^\s*(target|host|dbname|schema|type):|^\s{2,4}\w+:\s*(&\w+)?\s*$' | Where-Object { $_.Line -notmatch '(?i)password|secret|token' } | ForEach-Object { "$($_.LineNumber): $($_.Line.TrimEnd())" }
"===== production: postgres tables.yml group count and non-default sources ====="
$t = git show origin/production:extract_load/postgres/tables.yml
"target_schema entries: $(($t | Select-String 'target_schema:').Count)"
$t | Select-String -Pattern '^\s*-\s*source:\s*\S+' -Context 0,2 | ForEach-Object { ($_.Line + ' ' + ($_.Context.PostContext -join ' ')).Trim() }
"===== production: does postgres_extract_load have DBT_SELECT? ====="
git show origin/production:.gitlab-ci.yml | Select-String 'DBT_SELECT' | ForEach-Object { "$($_.LineNumber): $($_.Line.Trim())" }
