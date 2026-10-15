
GitLab CI/CD – Extract/Load Investigation Notes

Project: Office of Analytics
Working with: Josh
Date: October 14, 2026
Current focus: DEV Extract/Load implementation

1. Background

Josh wants us to get the existing Extract/Load pipeline working in DEV first. Once DEV is working and validated, we can move on to TEST and PROD.

The main success requirement is that 100% of the configured tables must extract and load successfully. Just having a green GitLab pipeline is not enough.

I spent time reviewing the GitLab YAML, Python Extract/Load modules, PostgreSQL configuration, GitLab variables, branch protection, verification logic, and existing pipeline behavior.

The repository already has a process for loading production-derived data into DEV and TEST. We should use that existing approach instead of redesigning the pipeline.

2. Current Architecture

The existing data flow appears to be:

Oracle / SQL Server / PostgreSQL sources
→ Production Extract/Load
→ PROD PostgreSQL raw tables
→ prod_mirror
→ DEV or TEST PostgreSQL raw tables

For now, focus only on DEV.

The upstream extraction jobs are configured for the production/default branch. The lower environments use the mirror job.

3. Files Reviewed

GitLab CI/CD

* .gitlab-ci.yml — Stages, jobs, runners, rules, variables, dependencies and failure behavior.

dbt

* dbt/profiles.yml — DEV PostgreSQL connection details and target configuration.

Mirror module

* src/extract_load/mirror/__main__.py — Entry point.
* src/extract_load/mirror/run.py — Mirror execution and safety checks.
* src/extract_load/mirror/connections.py — Production PostgreSQL source connection.
* src/extract_load/mirror/mappings.py — Mirror table mappings.

Shared Extract/Load

* src/extract_load/common/env.py — Environment configuration loading.
* src/extract_load/common/config.py — Required variables and environment resolution.
* src/extract_load/common/target.py — PostgreSQL target connection.
* src/extract_load/common/cli.py — Main Extract/Load execution.
* src/extract_load/common/state.py — State tracking and metadata.
* src/extract_load/common/verify.py — Source/target row-count verification.

These files were reviewed through VS Code screenshots and Microsoft 365 Copilot analysis. Runtime behavior has not yet been fully tested.

4. GitLab Pipeline Findings

Upstream Extract/Load

The repository has three main upstream jobs:

* oracle_extract_load
* sqlserver_extract_load
* postgres_extract_load

They inherit .el_upstream.

The reviewed rules include:

rules:
  - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    when: never
  - if: '$SKIP_EXTRACT_LOAD =~ /^(1|true|yes|on)$/i'
    when: never
  - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
    variables:
      DBT_TARGET: "prod"

These jobs are designed for the default branch, currently production.

We should not change their branch rules right now.

DEV Mirror

The prod_mirror job is the existing lower-environment loading mechanism.

It sets:

EL_MODULE: "mirror"

Execution command:

uv run python -m extract_load.mirror

The development-branch rules select DBT_TARGET=dev.

The job is manual by default, with an automatic execution condition involving PROD_MIRROR=true.

Although the job is named prod_mirror, the intended behavior is to read production PostgreSQL data and load it into a lower environment.

Environment Setup

env_setup prepares the Python/dbt environment.

Commands identified include:

uv sync
dbt deps
bl gen-refs

The runner tag is python-app05.

Runner execution and connectivity have not yet been tested.

Pipeline Failure Behavior

The shared .el_job template contains:

allow_failure: true

This means GitLab may tolerate an Extract/Load job failure and still allow the overall pipeline to complete.

Also, dbt_seed has optional Extract/Load dependencies.

For Josh’s 100% requirement, we need to validate the individual Extract/Load job result and all expected table outcomes, not just the overall pipeline status.

5. Mirror Execution Flow

The mirror entry point is:

src/extract_load/mirror/__main__.py

It imports main() from mirror/run.py.

Execution flow:

1. GitLab starts prod_mirror.
2. Python executes extract_load.mirror.
3. mirror.run.main() starts.
4. load_el_env() loads configuration.
5. _missing_env_vars() checks source variables.
6. _guard_target() checks the source and target.
7. run_el() starts the shared Extract/Load workflow.
8. Table mappings are loaded and selected.
9. Source and target connections are opened.
10. Verification or loading runs.
11. Table outcomes and exit status are returned.

The mirror passes the following functions into run_el():

* get_connection — Source PostgreSQL connection.
* extract_batches — Source data extraction.
* count_rows — Source row counts.
* load_mirror_mappings — Mirror table mappings.

Because run_el() is shared, changes to it may affect other Extract/Load modules.

6. Mirror Safety Checks

Reviewed _guard_target() in mirror/run.py.

The code:

* Rejects DBT_TARGET=prod.
* Checks whether source and target have the same host/database combination.
* Resolves the target from the dbt profile.
* Resolves the source through source_endpoint().

These are useful safeguards.

However, we still need to verify that the configured DEV endpoint is actually the approved DEV database.

7. DEV PostgreSQL Configuration

Reviewed dbt/profiles.yml.

Findings:

* Profile: business_layer
* Target: dev
* Database: PostgreSQL
* Port: 5432
* Schema: business_layer_dev
* Username: DBT_PG_USER
* Password: DBT_PG_PASSWORD
* SSL mode: require
* Threads: 4

DEV host and database are configured in the profile.

TEST and PROD configuration changes are outside the current scope.

Extract/Load Target Connection

src/extract_load/common/target.py uses:

endpoint = target_endpoint()
user = os.environ["TARGET_PG_USER"]
conn = psycopg.connect(
    host=endpoint.host,
    port=endpoint.port,
    user=user,
    password=os.environ["TARGET_PG_PASSWORD"],
    dbname=endpoint.dbname,
    autocommit=False,
)

Important finding:

dbt uses DBT_PG_USER and DBT_PG_PASSWORD.

Extract/Load uses TARGET_PG_USER and TARGET_PG_PASSWORD.

These variables are not automatically interchangeable.

We may be able to reuse the existing DEV PostgreSQL account, but database permissions and approval need to be confirmed.

8. GitLab Branch Protection

Checked GitLab Settings → Repository → Branch rules.

Protected branches include:

* production
* development
* test
* remote-development

The development branch is protected.

This is important because the existing DEV database variables are also protected.

Protected variables should be eligible for jobs on the protected development branch when the environment scope and pipeline context match.

A new feature branch will not automatically receive protected variables unless its protection and applicable GitLab rules allow them.

Do not disable variable protection to make testing work.

9. GitLab Variables

Checked project-level and inherited group-level variables.

Existing Project Variables

DBT_PG_USER

* Present.
* Protected.
* Environment scope: dev.

DBT_PG_PASSWORD

* Present.
* Protected.
* Masked/hidden.
* Environment scope: dev.

Seven inherited group variables were visible, mainly related to Python package configuration, Azure, and Tableau.

The required Extract/Load variables were not visible in those lists.

Required Production Mirror Source Variables

Defined in mirror/connections.py:

MIRROR_ENV_VARS = [
    "MIRROR_PG_HOST",
    "MIRROR_PG_DATABASE",
    "MIRROR_PG_USER",
    "MIRROR_PG_PASSWORD",
]

Optional:

MIRROR_PG_PORT — Defaults to 5432.

These connect to the production PostgreSQL source.

The source connection executes:

SET default_transaction_read_only = on;

The source account should also have database-enforced read-only permissions.

Required DEV Target Variables

Defined in common/config.py:

PG_ENV_VARS = [
    "TARGET_PG_USER",
    "TARGET_PG_PASSWORD",
]

These are required for the Extract/Load target connection.

The code returns exit code 1 if required variables are missing.

Current Variable Status

* DBT_PG_* — Configured.
* TARGET_PG_* — Not visible in GitLab project/group variables.
* MIRROR_PG_* — Not visible.
* EL_ENV_FILE — Not visible.
* Runtime availability from other sources — Not verified.

10. Environment File Investigation

Reviewed src/extract_load/common/env.py.

The loader supports:

1. Explicit EL_ENV_FILE.
2. Repository-level extract_load.env.
3. Existing process environment variables.

The code uses:

load_dotenv(dotenv_path=path, override=False)

This means existing environment variables take precedence over the dotenv file.

If no fallback environment file exists, the application can continue using process environment variables.

If EL_ENV_FILE explicitly points to a missing file, it raises an error.

Conclusion:

EL_ENV_FILE is not mandatory if the required configuration is supplied directly through GitLab or another approved process environment source.

We should not create unnecessary duplicate credential storage.

resolve_env()

Reviewed common/config.py.

Variable lookup supports profile-specific names before standard names.

Example:

MIRROR_PG_USER_PROD can take precedence over MIRROR_PG_USER when a prod profile is explicitly requested.

The mirror connection currently defaults to the standard variable names for the reviewed call path.

11. Important Code Issue — Verify-Only Mode

This is the main code issue identified during the investigation.

In src/extract_load/common/cli.py, the current execution order is:

pool = ConnectionPool(connect_source)
pg_conn = pool.pg()
ensure_state_table(pg_conn)
if args.verify_only:
    # Verification logic
    ...

ensure_state_table() runs before the verification branch.

Reviewed common/state.py.

The function executes statements including:

CREATE SCHEMA IF NOT EXISTS extract_load_meta;
CREATE TABLE IF NOT EXISTS extract_load_meta._el_state (...);
ALTER TABLE extract_load_meta._el_state
ADD COLUMN IF NOT EXISTS ...;

It also commits.

Therefore, --verify-only is not guaranteed to be read-only.

Even though these statements use IF NOT EXISTS, they can modify the database if objects or columns are missing.

Proposed Fix

Move:

ensure_state_table(pg_conn)

below the if args.verify_only: branch.

Expected structure:

pool = ConnectionPool(connect_source)
pg_conn = pool.pg()
if args.verify_only:
    # Compare source and target counts
    # Return verification result
    ...
ensure_state_table(pg_conn)
# Continue normal Extract/Load

This is a proposed change, not an implemented patch.

Before applying it, confirm that verification does not require the state table and that other connection/helper functions do not perform writes.

This change is in shared code, so regression testing is required for other Extract/Load modules.

12. Verification Logic

Reviewed common/verify.py and _run_verification() in common/cli.py.

Verification compares source and target row counts.

Possible results:

* match — Counts agree.
* under — Target count is lower.
* missing — Target table does not exist.
* error — Comparison failed.
* over — Target count is higher.

Exit code 1 is returned for under, missing, or error.

over is treated as a warning under the current verification rules.

An optional JSON report can be generated.

The count-comparison functions reviewed did not show database writes. The confirmed DDL concern is the earlier state-table initialization.

Row-count verification is useful but does not prove complete data accuracy.

13. State Tracking and Failure Handling

The loader uses the metadata state table:

extract_load_meta._el_state

It supports state tracking for incremental processing and reruns.

Normal Extract/Load can perform operations including:

* Inserts.
* Full refreshes.
* Deletes/replacements for incremental windows.
* Table creation or recreation.
* State updates.

The exact operations depend on the selected mappings and loading mode.

Per-table failures are collected and can cause the Python application to exit with a nonzero status.

However, GitLab’s allow_failure: true may prevent that failure from making the overall pipeline fail.

For Josh’s requirement, check:

* Expected table count.
* Selected table count.
* Successful table count.
* Failed tables.
* Unexpected skipped tables.
* Target data validation.
* Job exit status.

Expected tables, selected tables and successful tables must match for the approved run.

Historical state-table failures should not be confused with failures from the current execution.

14. Credentials Needed

For offline development:

No database credentials are needed.

For actual DEV mirror testing:

Production source

* MIRROR_PG_HOST
* MIRROR_PG_DATABASE
* MIRROR_PG_USER
* MIRROR_PG_PASSWORD
* Optional MIRROR_PG_PORT

The source account should be read-only.

DEV target

* TARGET_PG_USER
* TARGET_PG_PASSWORD

The target account needs approved permissions for the required loading operations.

Existing DEV DBT_PG_* credentials are already configured but may not have the necessary Extract/Load permissions.

We do not need production credentials to create the feature branch, fix code, or run offline validation.

We do need approved production read-only access before running the real production-to-DEV mirror.

15. Information Needed from Ben

Need confirmation of:

1. Whether the existing DEV PostgreSQL dbt account can be used for Extract/Load.
2. Whether it has appropriate write permissions on the DEV raw schemas.
3. Whether it can create or update objects in extract_load_meta.
4. Whether a production read-only PostgreSQL mirror account already exists.
5. Whether those source credentials are managed through an approved secret system.
6. Whether the configured DEV target endpoint is correct.

Do not request passwords through normal Teams messages or store them in repository files.

16. Development Strategy

The next step is to create a feature branch from development.

Suggested branch:

feature/dev-el-pipeline-fix

Before creating it:

git status
git branch --show-current

If the working tree is clean:

git switch development
git pull origin development
git switch -c feature/dev-el-pipeline-fix

Initial Work

1. Prepare the minimal --verify-only correction in common/cli.py.
2. Add a unit test confirming verification skips ensure_state_table().
3. Check that normal Extract/Load still initializes the state table.
4. Run Python syntax checks and relevant tests.
5. Validate GitLab YAML.
6. Review feature-branch pipeline rules.

The existing prod_mirror rules are branch-specific. The feature branch may not include the normal mirror job.

We may need a temporary, explicitly gated test mechanism that cannot accidentally perform a production or DEV data load.

Do not change protected-branch rules or expose protected variables merely to run feature-branch tests.

Before Actual DEV Execution

Confirm:

* Correct DEV target.
* Approved production source.
* Required variables.
* Database permissions.
* Runner connectivity.
* Selected table mappings.
* Load mode.
* Data-modification behavior.
* Authorization for the test.

Then perform a controlled DEV execution.

After Execution

Validate:

* Job result.
* Expected versus selected tables.
* Per-table outcomes.
* Source/target row counts.
* State records.
* Any failed or skipped tables.
* Relevant data consistency checks.

Prepare a merge request after successful testing and review.

17. Current Status

Completed investigation:

* GitLab branch rules reviewed.
* Protected DEV branch confirmed.
* DEV dbt profile reviewed.
* Existing DEV dbt variables confirmed.
* Mirror execution path traced.
* Source and target connection functions reviewed.
* Environment loading reviewed.
* Required variable names identified.
* Verification and failure handling reviewed.
* Verify-only DDL issue identified.

Not completed:

* Feature branch creation.
* Code modification.
* Unit tests.
* Runner connectivity test.
* Production source credential confirmation.
* DEV target write-permission confirmation.
* Read-only preflight.
* Actual DEV mirror execution.
* 100% table-load validation.
* Merge request.

18. Things We Should Not Reinvestigate

The following have already been established sufficiently for the next development phase:

* Josh wants DEV working first.
* TEST and PROD changes are deferred.
* The existing lower-environment mechanism is prod_mirror.
* Upstream Oracle, SQL Server, and PostgreSQL jobs are default-branch oriented.
* DEV dbt credentials already exist in GitLab.
* development is protected.
* EL_ENV_FILE is optional.
* Extract/Load requires TARGET_PG_*.
* Mirror requires MIRROR_PG_*.
* Mirror source is intended to be read-only.
* Verify-only currently performs metadata initialization before verification.
* The shared loader can return a failure status.
* GitLab’s allow_failure: true means overall pipeline success is insufficient.

Only revisit these findings if new code, configuration, or runtime evidence contradicts them.

19. Where to Resume

Current stopping point:

The code investigation is complete enough to begin implementation.

No code changes or database tests have been performed yet.

Next working session:

1. Check local Git status.
2. Create the feature branch from development.
3. Implement and unit-test the minimal verification-path fix.
4. Validate CI configuration.
5. Confirm the required DEV and production-source access.
6. Prepare the first approved DEV test.

The immediate goal is not another repository analysis. It is getting a reviewed change ready, resolving credential dependencies, and performing a controlled DEV validation.

**
