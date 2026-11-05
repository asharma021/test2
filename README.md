& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
$t = git show origin/development:src/extract_load/postgres/tables.yml
"===== 1. tables.yml entries for the failing tables ====="
$names = 'message_details','message_thread','message_tracker','notification','bank_account','export_certificates','export_products'
$i = 0; foreach ($ln in $t) { $i++; foreach ($n in $names) { if ($ln -match "source_table:\s*$n\b") { $s = [Math]::Max(0,$i-1); $t[$s..([Math]::Min($s+6,$t.Count-1))] | ForEach-Object { "{0,5}: {1}" -f ($s+1), $_ }; "---" } } }
"===== 2. Shared incremental defaults (anchors) ====="
$t | Select-String -Pattern '&\w+|unique_key' | ForEach-Object { "{0,5}: {1}" -f $_.LineNumber, $_.Line }
"===== 3. How the mirror picks the key / index column ====="
git grep -n -E 'unique_key|_id_idx|"id"|\bid\b' origin/development -- src/extract_load/mirror src/extract_load/common/config.py src/extract_load/common/target.py | Select-Object -First 25
"===== 4. dbt sources/staging that describe these tables (to find real key columns) ====="
git grep -n -i -E 'export_certificates|export_products|myttb_message_details|myttb_bank_account|myttb_notification' origin/development -- dbt/models | Select-Object -First 25
}
