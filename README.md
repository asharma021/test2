Set-Location C:\Users\SharmaAashishR\oa-business-layer

# 1. Which commit changed postgres_extract_load (audit trail)
git log --oneline origin/development..origin/pipeline-testing -L '/^postgres_extract_load:/,+30:.gitlab-ci.yml' --no-patch

# 2. Does development's file have the spec/inputs '---' separator? (diff showed pipeline-testing added one)
git show "origin/development:.gitlab-ci.yml" | Select-String -Pattern '^---\s*$' |
  ForEach-Object { "development '---' at line $($_.LineNumber)" }
git show "origin/pipeline-testing:.gitlab-ci.yml" | Select-String -Pattern '^---\s*$' |
  ForEach-Object { "pipeline-testing '---' at line $($_.LineNumber)" }

# 3. What is the untracked backup file vs the committed file?
git diff --no-index --stat -- .gitlab-ci.yml .gitlab-ci.yml.backup
