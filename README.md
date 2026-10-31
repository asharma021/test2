Set-Location C:\Users\SharmaAashishR\oa-business-layer
$f = (Get-ChildItem src\extract_load -Recurse -Filter config.py | Select-Object -First 1).FullName
$L = Get-Content $f
& {
  "===== config.py 28-50 ====="
  for ($i = 28; $i -le 50; $i++) { "{0,4}: {1}" -f $i, $L[$i-1] }
  "===== config.py 150-175 ====="
  for ($i = 150; $i -le 175; $i++) { "{0,4}: {1}" -f $i, $L[$i-1] }
} | Out-File -Encoding utf8 (Join-Path $env:TEMP 'el_strategy.txt')
notepad (Join-Path $env:TEMP 'el_strategy.txt')
