Hi Josh, I'm working on the PostgreSQL TEST validation. DEV verification passed successfully, but I noticed the TEST connection in `dbt/profiles.yml` still has placeholder values.

Could you confirm the correct TEST PostgreSQL hostname, database name, and schema, and whether the TEST credentials are already configured for the GitLab runner?

Once confirmed, I'll proceed with the TEST verify-only validation.
