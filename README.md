    - if: '$CI_COMMIT_BRANCH == "pipeline-testing" && $CI_PIPELINE_SOURCE == "web" && $DBT_TARGET == "test"'
      variables:
        DBT_TARGET: "test"
        EL_VERIFY_ONLY: "true"
        DBT_SELECT: "myttb_cg_ref_identifier_type"
        DBT_LOAD_MODE: "incremental"
      when: manual
