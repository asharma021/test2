Select-String -Path ".gitlab-ci.yml" `
  -Pattern '^\s*\.el_job:|^\s*\.el_upstream:|EL_MODULE|EL_ARGS|uv run|extract-load' `
  -Context 3,12 |
  Select-Object -First 100


  
