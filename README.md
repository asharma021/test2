check_postgres_test_access:
  stage: extract_load
  tags:
    - python-app05
  rules:
    - if: '$CI_COMMIT_BRANCH == "pipeline-testing" && $CI_PIPELINE_SOURCE == "web"'
      when: manual
    - when: never
  script:
    - |
      $testDir = '\\tthqpcelerra01\ttb_common\OA_Data_Sharing\OA\dbt_test_env'
      $testFile = Join-Path $testDir 'extract_load.env'

      Write-Host "Checking TEST environment access from GitLab runner"

      if (Test-Path -LiteralPath $testDir -PathType Container) {
        Write-Host "TEST directory: ACCESSIBLE"
      } else {
        Write-Host "TEST directory: NOT ACCESSIBLE"
      }

      if (Test-Path -LiteralPath $testFile -PathType Leaf) {
        Write-Host "TEST environment file: ACCESSIBLE"
      } else {
        Write-Host "TEST environment file: NOT ACCESSIBLE"
      }


git diff --check
git diff -- .gitlab-ci.yml
git status --short



git add .gitlab-ci.yml
git commit -m "Add PostgreSQL TEST runner access diagnostic"
git push -o ci.skip origin pipeline-testing
