Get-Content -LiteralPath $path |
  Where-Object {
    $_ -match '^\s*SRC_PG_(USER|PASSWORD|HOST|DATABASE)(?:_IVV)?\s*='
  } |
  ForEach-Object {
    ($_ -split '=', 2)[0].Trim()
  }
