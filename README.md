postgres_extract_load_test:
  extends: .el_job
  stage: extract_load
  tags:
    - python-app05
  variables:
    EL_MODULE: "postgres"
    EL_LABEL: "PostgreSQL IVV Verify-Only"
    EL_ENV_FILE: '\\tthqpcelerra01\ttb_common\OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'
    DBT_TARGET: "test"
    EL_VERIFY_ONLY: "true"
    DBT_SELECT: "myttb_cg_ref_identifier_type"
    DBT_LOAD_MODE: "incremental"
  rules:
    - if: '$CI_COMMIT_BRANCH == "pipeline-testing" && $CI_PIPELINE_SOURCE == "web"'
      when: manual
    - when: never
  before_script:
    - uv sync --quiet
    - |
      if (-not (Test-Path -LiteralPath $env:EL_ENV_FILE -PathType Leaf)) {
          throw "Approved Extract/Load environment file is not accessible"
      }

      $required = @(
          "SRC_PG_HOST_IVW",
          "SRC_PG_PORT_IVW",
          "SRC_PG_DATABASE_IVW"
      )

      $values = @{}
      foreach ($line in [System.IO.File]::ReadAllLines($env:EL_ENV_FILE)) {
          if ($line -match '^\s*(SRC_PG_(?:HOST|PORT|DATABASE)_IVW)\s*=\s*(.*)\s*$') {
              $values[$matches[1]] = $matches[2].Trim().Trim('"', "'")
          }
      }

      foreach ($key in $required) {
          if (-not $values.ContainsKey($key) -or [string]::IsNullOrWhiteSpace($values[$key])) {
              throw "Missing required IVW source setting: $key"
          }
      }

      $env:SRC_PG_HOST = $values["SRC_PG_HOST_IVW"]
      $env:SRC_PG_PORT = $values["SRC_PG_PORT_IVW"]
      $env:SRC_PG_DATABASE = $values["SRC_PG_DATABASE_IVW"]

      Write-Host "IVW PostgreSQL source host, port and database selected"
