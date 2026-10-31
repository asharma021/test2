Set-Location C:\Users\SharmaAashishR\oa-business-layer
$f = (Get-ChildItem src\extract_load -Recurse -Filter sync.py | Select-Object -First 1).FullName
$L = Get-Content $f
$s = ($L | Select-String -Pattern '^def _sync_incremental').LineNumber
$out = Join-Path $env:TEMP 'el_incremental.txt'
& {
  "===== sync.py _sync_incremental (from line $s) ====="
  for ($i = $s; $i -le [Math]::Min($s + 130, $L.Count); $i++) { "{0,4}: {1}" -f $i, $L[$i-1] }
  "===== unique_key usage ====="
  Select-String -Path (Get-ChildItem src\extract_load -Recurse -Filter *.py).FullName -Pattern 'unique_key' |
    ForEach-Object { "{0}:{1}: {2}" -f $_.Filename, $_.LineNumber, $_.Line.Trim() }
} | Out-File -Encoding utf8 $out
notepad $out
