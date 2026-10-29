
# Guard for approved non-production PostgreSQL load jobs
.postgres_incremental_guard:
  before_script:
    - |
      $ErrorActionPreference = "Stop"

      if ($env:CI_COMMIT_REF_PROTECTED -ne "true") {
        throw "Protected branch required"
      }

      if ($env:CI_COMMIT_BRANCH -ne "pipeline-testing") {
        throw "Incremental loading restricted to pipeline-testing"
      }

      if ($env:CI_PIPELINE_SOURCE -ne "web") {
        throw "Manual web pipeline required"
      }

      if ($env:EL_VERIFY_ONLY -ne "false") {
        throw "Unexpected verification mode"
      }

      if ($env:DBT_LOAD_MODE -ne "incremental") {
        throw "Unexpected load mode"
      }

      if ([string]::IsNullOrWhiteSpace($env:EL_CONFIG)) {
        throw "Approved mapping file is required"
      }

      if ([string]::IsNullOrWhiteSpace($env:DBT_SELECT)) {
        throw "Approved table selection is required"
      }

      if ([string]::IsNullOrWhiteSpace($env:EL_ENV_FILE)) {
        throw "Environment configuration file is required"
      }

      if ($env:CI_ENVIRONMENT_NAME -eq "dev") {
        if ($env:DBT_TARGET -ne "dev") {
          throw "DEV environment mismatch"
        }
      }
      elseif ($env:CI_ENVIRONMENT_NAME -eq "ivv") {
        if ($env:DBT_TARGET -ne "test") {
          throw "IVV environment mismatch"
        }
      }
      else {
        throw "Unsupported environment"
      }

      if (-not (Test-Path -LiteralPath $env:EL_CONFIG -PathType Leaf)) {
        throw "Mapping file not found"
      }

      if (-not (Test-Path -LiteralPath $env:EL_ENV_FILE -PathType Leaf)) {
        throw "Environment file not accessible"
      }

      Write-Host "PostgreSQL incremental prechecks passed"
    - uv sync --quiet
