check_python_app05_access:
  stage: extract_load
  tags:
    - python-app05
  rules:
    - when: manual
  script:
    - |
      whoami

      $path = '\\thqpcelerra01\ttb_common\_OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'

      Write-Host "Checking UNC access from python-app05"

      if (Test-Path -LiteralPath $path -PathType Leaf) {
          Write-Host "SUCCESS: File accessible"
      } else {
          Write-Host "FAILED: File not accessible"
      }
