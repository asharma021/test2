postgres_extract_load:
  extends: .el_upstream
  variables:
    EL_MODULE: "postgres"
    EL_LABEL: "PostgreSQL Extract/Load"

  rules:
    - if: '$CI_COMMIT_BRANCH == "feature/postgres-el-cicd-testing" && $CI_PIPELINE_SOURCE == "web"'
      variables:
        DBT_TARGET: "dev"
        EL_VERIFY_ONLY: "true"
        DBT_SELECT: "myttb_cg_ref_identifier_type"
        DBT_LOAD_MODE: "incremental"
      when: manual
