$ErrorActionPreference = "Stop"

try {
    if ($env:CI_COMMIT_REF_PROTECTED -ne "true") {
        throw "Protected branch required"
    }

    if ($env:CI_COMMIT_BRANCH -ne "pipeline-testing") {
        throw "Incorrect branch"
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

    if ($env:CI_ENVIRONMENT_NAME -eq "ivv") {
        if ($env:DBT_TARGET -ne "test") {
            throw "IVV environment mismatch"
        }
    }
    else {
        throw "Unsupported test environment"
    }

    Write-Host "PASS: PostgreSQL IVV guard checks passed"
}
catch {
    Write-Host "FAIL: $($_.Exception.Message)"
}
