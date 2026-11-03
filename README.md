& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
"===== A. Umair's commit 5204ee3: files ====="
git show --stat --format='%h %an : %s' 5204ee3
"===== B. Umair's commit: extract/load config diff ====="
git --no-pager show --format='' 5204ee3 -- src/extract_load | Select-Object -First 120
"===== C. Who changed tables_ivv_verify.yml on development (ever) ====="
git log --format='%h %ad %an : %s' --date=short origin/development -- src/extract_load/postgres/tables_ivv_verify.yml
"===== D. The 'incremental -> full' permitmod commit ====="
$c = (git log --format='%h' -1 --grep='permitmod' origin/development)
if ($c) { git show --stat --format='%h %an : %s' $c; git --no-pager show --format='' $c -- src/extract_load | Select-Object -First 60 } else { "(not found)" }
}
