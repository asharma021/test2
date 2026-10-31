Set-Location C:\Users\SharmaAashishR\oa-business-layer
$f = (Get-ChildItem src\extract_load -Recurse -Filter config.py | Select-Object -First 1).FullName
Select-String -Path $f -Pattern 'INCREMENTAL_STRATEGIES|strategy' |
  ForEach-Object { "{0}: {1}" -f $_.LineNumber, $_.Line.Trim() }
