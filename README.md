Set-Location C:\Users\SharmaAashishR\oa-business-layer
git add -- .gitlab-ci.yml
git status --short          # expect: M  .gitlab-ci.yml  (staged) and ?? .gitlab-ci.yml.backup
git commit -m "Restore approved postgres_extract_load block from development" -m "Removes test-only job-level variables (IVV mapping, verify-only, DBT_TARGET=test, single-table select) so they cannot reach the default-branch run. DEV verify-only stays manual on pipeline-testing/development. IVV verify (postgres_extract_load_test) and DEV/IVV incremental jobs unchanged."


git push -o ci.skip origin pipeline-testing



Set-Location C:\Users\SharmaAashishR\oa-business-layer
Select-String -Path src\extract_load\common\*.py, src\extract_load\postgres\*.py `
  -Pattern 'fnmatch|startswith\(|def \w*select\w*|load_state|checkpoint|fuzz|full_refresh|_el_' |
  ForEach-Object { "{0}:{1}: {2}" -f $_.Filename, $_.LineNumber, $_.Line.Trim() }
