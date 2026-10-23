  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: never

    - if: '$[[ inputs.skip_extract_load ]] == "true"'
      when: never

    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      variables:
        DBT_TARGET: "prod"

    - if: '($CI_COMMIT_BRANCH == "pipeline-testing" || $CI_COMMIT_BRANCH == "development") && $CI_PIPELINE_SOURCE == "web"'
      variables:
        DBT_TARGET: "dev"
        EL_VERIFY_ONLY: "true"
        DBT_SELECT: "myttb_cg_ref_identifier_type"
        DBT_LOAD_MODE: "incremental"
      when: manual
