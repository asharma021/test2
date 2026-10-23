check_postgres_test_config:
  stage: extract_load
  tags:
    - python-app05
  rules:
    - if: '$CI_COMMIT_BRANCH == "pipeline-testing" && $CI_PIPELINE_SOURCE == "web"'
      when: manual
  allow_failure: true
  script:
    - |
      Write-Host "Checking TEST environment configuration"
      Write-Host "Runner identity: $(whoami)"

      $testFile = "\\tthqpcelerra01\ttb_common\OA_Data_Sharing\OA\dbt_test_env\extract_load.env"

      if (Test-Path -LiteralPath $testFile -PathType Leaf) {
        Write-Host "TEST environment file: ACCESSIBLE"
      } else {
        Write-Host "TEST environment file: NOT ACCESSIBLE"
      }
