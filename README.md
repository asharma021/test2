check_el_file_access:
  stage: extract_load
  tags:
    - python-app05
  rules:
    - if: '$CI_COMMIT_BRANCH == "pipeline-testing" && $CI_PIPELINE_SOURCE == "web"'
      when: manual
    - when: never
  script:
    - |
      $path = '\\thqpcelerra01\ttb_common\OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'
      if (Test-Path -LiteralPath $path -PathType Leaf) {
        Write-Output "EL environment file is accessible"
      } else {
        Write-Output "EL environment file is not accessible"
        exit 1
      }
