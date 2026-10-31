Set-Location C:\Users\SharmaAashishR\oa-business-layer
function Show-Lines([string]$name, [int]$a, [int]$b) {
  $f = (Get-ChildItem src\extract_load -Recurse -Filter $name | Select-Object -First 1).FullName
  "===== $name ($f) lines $a-$b ====="
  $L = Get-Content $f
  for ($i = $a; $i -le [Math]::Min($b, $L.Count); $i++) { "{0,4}: {1}" -f $i, $L[$i-1] }
}
$out = Join-Path $env:TEMP 'el_review.txt'
& {
  Show-Lines selection.py 1 120
  Show-Lines sync.py 275 340
  Show-Lines state.py 270 300
  Show-Lines target.py 570 600
  Select-String -Path (Get-ChildItem src\extract_load -Recurse -Filter *.py).FullName -Pattern 'DROP TABLE|CASCADE' |
    ForEach-Object { "{0}:{1}: {2}" -f $_.Filename, $_.LineNumber, $_.Line.Trim() }
} | Out-File -Encoding utf8 $out
notepad $out
