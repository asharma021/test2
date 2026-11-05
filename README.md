& {
$zip = Get-ChildItem "$env:USERPROFILE\Downloads" -Filter 'artifacts*.zip' | Sort-Object LastWriteTime -Descending | Select-Object -First 1
if (-not $zip) { "No artifacts zip in Downloads"; return }
$out = Join-Path $env:TEMP 'oa_mirror_log'
Remove-Item $out -Recurse -Force -ErrorAction SilentlyContinue
Expand-Archive $zip.FullName $out
$log = Get-ChildItem $out -Recurse -Filter 'prod_mirror.log' | Select-Object -First 1
"===== myttb_controlled_group ====="
Select-String -Path $log.FullName -Pattern 'myttb_controlled_group failed' -Context 0,3 | ForEach-Object { $_.Line; $_.Context.PostContext }
"===== export_certificates ====="
Select-String -Path $log.FullName -Pattern 'export_certificates failed' -Context 0,3 | ForEach-Object { $_.Line; $_.Context.PostContext }
"===== Distinct error reasons (top 10) ====="
Select-String -Path $log.FullName -Pattern 'failed after .*?: (.*)$' | ForEach-Object { ($_.Matches[0].Groups[1].Value -replace '"[^"]*"', '"X"') } | Group-Object | Sort-Object Count -Descending | Select-Object -First 10 | ForEach-Object { "{0,4} x {1}" -f $_.Count, $_.Name }
}
