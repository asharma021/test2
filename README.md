Configured the PostgreSQL Extract/Load DEV verification job on `feature/postgres-el-cicd-testing`.

- Updated `.gitlab-ci.yml` with a manual DEV verification job.
- Committed and pushed changes to GitLab.
- Validated pipeline configuration and confirmed successful pipeline creation.
- Confirmed DEV PostgreSQL credentials are configured in GitLab CI/CD variables.
- Triggered pipeline #67654, but the job remains pending because runner #766 (`python-app05`) is protected and cannot execute jobs from the unprotected feature branch.

**Current blocker:** Runner access for feature-branch testing.

**Next step:** Coordinate with Josh to confirm the approved runner configuration, then execute DEV verification and review the results. No changes have been made to production.
