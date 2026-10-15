
I already have DEV PostgreSQL credentials (DBT_PG_USER and DBT_PG_PASSWORD) and DEV Oracle username/password. My immediate priority is to get the existing Oracle-to-PostgreSQL Extract/Load pipeline working in DEV with Josh.

Inspect .gitlab-ci.yml, extract_load/oracle/run.py, extract_load/oracle/connections.py, extract_load/common/cli.py, extract_load/common/env.py, and the DEV dbt profile.

Determine: 1. Which GitLab job executes Oracle Extract/Load? 2. Which Oracle connection variables are required? 3. Does the target use DBT_PG_USER/PASSWORD or TARGET_PG_USER/PASSWORD? 4. Are those names mapped? 5. What non-secret connection settings are required? 6. Can the job access credentials given branch/environment protection? 7. What safe preflight checks are possible?

Cite exact file/function references. Separate confirmed requirements from missing information. Do not assume prod_mirror is the correct job. Do not request passwords or recommend execution yet. End with one next action.
