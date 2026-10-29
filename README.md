
Get-ChildItem src/extract_load -Recurse -Filter "*.py" |
  Select-String -Pattern "ensure_state_table|incremental|upsert|checkpoint|watermark" |
  Select-Object Path, LineNumber, Line
