Select-String -Path .gitlab-ci.yml `
  -Pattern '^\s*\.el_upstream:' `
  -Context 0,65


  Select-String -Path .gitlab-d.yml `
  -Pattern '^\s*\.el_upstream:' `
  -Context 0,65
