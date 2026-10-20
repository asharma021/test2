postgres_extract_load:
  extends: .el_upstream
  variables:
    EL_MODULE: "postgres"
    EL_LABEL: "PostgreSQL Extract/Load"
    SRC_PG_USER: "$DBT_PG_USER"
    SRC_PG_PASSWORD: "$DBT_PG_PASSWORD"

  rules:
    - if: '$CI_COMMIT_BRANCH == "pipeline-testing" && $CI_PIPELINE_SOURCE == "web"'
      variables:
        DBT_TARGET: "dev"
        EL_VERIFY_ONLY: "true"
        DBT_SELECT: "myttb_cg_ref_identifier_type"
        DBT_LOAD_MODE: "incremental"
      when: manual
