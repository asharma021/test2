Select-String -Path .gitlab-ci.yml `
  -Pattern '^\s*\.el_job:' `
  -Context 0,85
