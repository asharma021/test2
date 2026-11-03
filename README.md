& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
"1) Files in Umair's commit:"; git show --name-only --format='' 5204ee3
"2) Commits that touched tables_ivv_verify.yml:"; git log --format='   %h %an : %s' origin/development -- src/extract_load/postgres/tables_ivv_verify.yml
"3) tableau_views_stats still in tables.yml on development: $([bool](git show origin/development:src/extract_load/postgres/tables.yml | Select-String -SimpleMatch 'tableau_views_stats' -Quiet))"
"4) 'incremental' lines added by Umair: $((git show --format='' 5204ee3 | Select-String -Pattern '^\+.*incremental').Count)"
"5) permitmod commit:"; git log -1 --format='   %h %an : %s' --grep='permitmod' origin/development
}
