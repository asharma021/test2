Josh confirmed our priority is to get the existing Extract/Load workflow working in DEV first. TEST and PROD changes come later. The existing prod_mirror is the lower-environment loading mechanism.

GitLab currently shows DBT_PG_USER and DBT_PG_PASSWORD protected and scoped to dev. I also have DEV PostgreSQL and Oracle credentials, but do not assume they are suitable for mirror source.

Inspect .gitlab-ci.yml, extract_load/mirror/run.py, extract_load/mirror/connections.py, extract_load/common/env.py, extract_load/common/cli.py, and dbt DEV profile.

1. Identify exact required DEV prod_mirror variables.
2. Which come from EL_ENV_FILE versus direct GitLab variables?
3. Can existing DBT_PG_USER/PASSWORD be reused as TARGET_PG_USER/PASSWORD subject to permission checks?
4. Check environment scope and protected branch variable availability.
5. Verify DEV target cannot resolve to PROD.
6. Identify safe non-writing setup steps.
7. Produce DEV preflight checklist.
8. Identify only information Ben/database team must supply.
9. Define 100% selected-table completion validation.

Do not change code, expose secrets, disable protections, or execute jobs. Return one immediate GitLab action and remaining DEV implementation steps. Focus only on DEV.
