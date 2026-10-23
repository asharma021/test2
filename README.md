git diff --check
git diff -- .gitlab-ci.yml
git status --short



    - if: '($CI_COMMIT_BRANCH == "pipeline-testing" || $CI_COMMIT_BRANCH == "development") && $CI_PIPELINE_SOURCE == "web"'
