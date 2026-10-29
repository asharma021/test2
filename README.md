$env:CI_COMMIT_REF_PROTECTED = "true"
$env:CI_COMMIT_BRANCH = "pipeline-testing"
$env:CI_PIPELINE_SOURCE = "web"
$env:CI_ENVIRONMENT_NAME = "ivv"
$env:DBT_TARGET = "test"
$env:EL_VERIFY_ONLY = "false"
$env:DBT_LOAD_MODE = "incremental"

Write-Host "Guard test environment variables configured"
