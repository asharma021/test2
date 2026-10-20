**Purpose**

Configure the protected `pipeline-testing` branch to support manual PostgreSQL Extract/Load DEV verification.

**Changes**
- Updated the PostgreSQL EL job branch rule.
- Configured `DBT_TARGET=dev`.
- Enabled `EL_VERIFY_ONLY=true` for read-only row-count verification.
- Limited the test to `myttb_cg_ref_identifier_type`.
- Kept the job manually triggered.

**Testing**
- Local pre-commit checks passed.
- GitLab pipeline execution pending on the protected testing branch.

**Next step**

Merge into `pipeline-testing`, confirm DEV credentials and runner access, and execute the manual PostgreSQL verification job. No production deployment is intended.
