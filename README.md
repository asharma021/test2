Set-Location C:\Users\SharmaAashishR\oa-business-layer
git log --oneline -S 'postgres_extract_load_test:' origin/development..origin/pipeline-testing -- .gitlab-ci.yml
git show --stat --oneline a2d1ead 83360e0
git show HEAD:src/extract_load/postgres/tables_ivv_verify.yml
Select-String -Path src\extract_load\common\cli.py, src\extract_load\postgres\*.py -Pattern 'tables\.yml|--config|tables_ivv' |
  ForEach-Object { "{0}:{1}: {2}" -f $_.Filename, $_.LineNumber, $_.Line.Trim() }
Select-String -Path .gitlab-ci.yml -Pattern '"postgres_extract_load"|extends:\s*postgres_extract_load\b' |
  ForEach-Object { "{0,5}: {1}" -f $_.LineNumber, $_.Line.Trim() }
