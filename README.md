
We have established that the development pipeline contains manual jobs and prod_mirror invokes the Extract/Load mirror functionality. Focus exclusively on tracing the mirror execution path before making changes or running jobs.

Inspect the actual mirror module, connections.py, env.py, and run_el().

1. What exact Python module executes when EL_MODULE=mirror?
2. Trace the complete function execution sequence from GitLab to database operations.
3. Which source PostgreSQL host, database, and profile are selected?
4. Which target PostgreSQL host, database, and schema are selected when DBT_TARGET=dev?
5. Which variables are mandatory, optional, or profile-specific overrides?
6. Does mirror read production data and write into DEV? Cite exact code.
7. Where are table mappings configured and how are tables selected?
8. Does it insert, upsert, delete, truncate, or otherwise modify databases?
9. Trace --verify-only and confirm whether it genuinely prevents all writes.
10. What happens when one table fails? Does the process continue, fail the GitLab job, or incorrectly report success?
11. Which missing files prevent complete analysis?

Provide an execution flow and a table of confirmed findings with exact file/function references. Do not assume GitLab environment: dev proves the actual database destination. Do not modify code, create variables, or recommend running prod_mirror. End with the single next action needed to safely validate DEV Extract/Load.
