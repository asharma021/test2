Hi Yves, I've made good progress on the PostgreSQL Extract/Load GitLab CI/CD pipeline. I have the protected testing branch and Windows runner working, and successfully validated the Python environment-file loader using `EL_ENV_FILE`.

The remaining blocker is the actual PostgreSQL source configuration. The existing DEV Business Layer credentials (`DBT_PG_USER` / `DBT_PG_PASSWORD`) don't authenticate against the source database.

Josh mentioned that the dedicated Extract/Load credentials are maintained in a shared `extract_load.env` file. I tested access from the GitLab runner, and it can reach the `ttb_common` share, but not the `OA_Data_Sharing` directory where the file is expected.

Could you help with the following?

1. Confirm the correct location of the existing `extract_load.env` file.
2. Confirm whether runner `#766` (`python-app05`) should have access to that location.
3. If the runner cannot access it directly, help arrange for the approved configuration to be provided securely through a protected GitLab File-type variable named `EL_ENV_FILE`.
4. Confirm the intended PostgreSQL source host/database and service account for DEV verification.

No need to send passwords over Teams. Once the approved configuration is available, I can complete the single-table DEV verification for `myttb_cg_ref_identifier_type` and continue with the pipeline testing.

Everything else on the CI/CD and Python environment-loading side has been validated successfully.
