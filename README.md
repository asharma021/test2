Get-ChildItem -Path . -Recurse -File |
  Select-String -Pattern 'seed_abbreviation_standards' |
  Select-Object Path, LineNumber, Line
