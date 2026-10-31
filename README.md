Set-Location C:\Users\SharmaAashishR\oa-business-layer
$y = git show origin/pipeline-testing:.gitlab-ci.yml
foreach ($m in ($y | Select-String -Pattern 'EL_CONFIG:')) {
  $line = $m.Line
  $bad  = @($line.ToCharArray() | Where-Object { [int]$_ -gt 126 -or ([int]$_ -lt 32 -and [int]$_ -ne 9) } | ForEach-Object { 'U+{0:X4}' -f [int]$_ })
  $val  = ($line -replace '^\s*EL_CONFIG:\s*["'']?', '') -replace '["'']?\s*$', ''
  "{0,5}: {1}" -f $m.LineNumber, $line.Trim()
  "       hidden/non-ASCII chars: $(if ($bad) { $bad -join ' ' } else { 'none' })   value length: $($val.Length)   file exists as written: $(Test-Path -LiteralPath $val)"
}
'--- Exact filename characters in git ---'
git -c core.quotepath=true ls-tree --name-only origin/pipeline-testing src/extract_load/postgres/ | Select-String 'ivv'
