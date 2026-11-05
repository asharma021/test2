& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
"===== Josh's latest commits on development ====="
git log -3 --format='%h %ad %an : %s' --date=short origin/development
"===== Files changed in the latest commit ====="
git show --stat --format='' origin/development
"===== Active (not commented) groups in tables.yml ====="
git show origin/development:src/extract_load/postgres/tables.yml | Select-String -Pattern '^\s*-\s*(source:|source_schema:)' | ForEach-Object { $_.Line.Trim() }
"===== ITDS / Tableau lines still active? (expect none) ====="
git show origin/development:src/extract_load/postgres/tables.yml | Select-String -Pattern 'itds|tableau' | Where-Object { $_.Line -notmatch '^\s*#' } | ForEach-Object { "{0,5}: {1}" -f $_.LineNumber, $_.Line.Trim() }
}
