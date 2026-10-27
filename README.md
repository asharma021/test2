Select-String -Path .gitlab-ci.yml `
  -Pattern '^\s*\.el_upstream:' `
  -Context 0,95
