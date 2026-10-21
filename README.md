git branch --show-current
git status --short



check_el_env_loader:
  stage: extract_load
  tags:
    - python-app05
  rules:
    - if: '$CI_COMMIT_BRANCH == "pipeline-testing" && $CI_PIPELINE_SOURCE == "web"'
      when: manual
    - when: never
  script:
    - |
      $ErrorActionPreference = "Stop"

      $testFile = Join-Path $env:CI_PROJECT_DIR "el_loader_test.env"

      try {
        Set-Content -LiteralPath $testFile -Value @(
          "EL_TEST_MARKER=gitlab-loader-check"
        )

        $env:EL_ENV_FILE = $testFile

        if (-not (Test-Path -LiteralPath $env:EL_ENV_FILE -PathType Leaf)) {
          throw "Test environment file was not created"
        }

        Write-Output "Test environment file: ACCESSIBLE"

        uv run --no-sync python -c "from extract_load.common.env import load_el_env; import os; load_el_env(); assert os.getenv('EL_TEST_MARKER') == 'gitlab-loader-check'; print('Python EL environment loader: PASSED')"

        if ($LASTEXITCODE -ne 0) {
          throw "Python EL environment loader failed"
        }
      }
      finally {
        Remove-Item -LiteralPath $testFile -Force -ErrorAction SilentlyContinue
      }


git diff --check
git diff -- .gitlab-ci.yml



      
