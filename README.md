
TTB OFFICE OF ANALYTICS
POSTGRESQL EXTRACT/LOAD – PYTHON IMPLEMENTATION REVIEW

Project: OA Business Layer Modernization
Repository: oa-business-layer
Component: PostgreSQL Extract/Load (EL)
Working Branch: feature/postgres-el-cicd-testing
Document Type: Technical Review and Implementation Notes


1. OVERVIEW
------------

I reviewed the PostgreSQL Extract/Load Python implementation in the OA Business Layer repository to understand how the existing framework handles database connections, table mappings, extraction, loading, verification, and error handling.

The primary objective was to understand the Python execution flow before running the PostgreSQL EL process through GitLab CI/CD.

The repository already contains a shared Extract/Load framework supporting PostgreSQL, Oracle, SQL Server, and Informix.

Each source system has its own connection and extraction modules. The common Python package handles the shared execution logic.

The PostgreSQL implementation is already available. The current focus is validating its execution, resolving environment and dependency issues, and confirming that it can run successfully using the configured database connections.


2. PYTHON PROJECT STRUCTURE
----------------------------

The Extract/Load implementation is located under:

src/extract_load/

Directory structure:

src/
  extract_load/
    __init__.py
    README.md

    common/
      __init__.py
      cli.py
      config.py
      env.py
      target.py
      verify.py
      selection.py
      state.py
      sync.py

    postgres/
      __init__.py
      __main__.py
      connections.py
      extract.py
      run.py
      tables.yml

    oracle/
      __init__.py
      connections.py
      run.py
      tables.yml

    sqlserver/
      __init__.py
      connections.py
      extract.py
      run.py

    informix/
      __init__.py
      __main__.py
      connections.py
      extract.py
      run.py
      tables.yml

    mirror/
      __init__.py
      connections.py
      mappings.py
      run.py

The source-specific directories contain the database connection and extraction implementations.

The common directory contains shared logic used across the different source systems.


3. POSTGRESQL PYTHON FILES
---------------------------

3.1 connections.py – Source Database Connection

Location:
src/extract_load/postgres/connections.py

Purpose:
Establishes a connection to the source PostgreSQL database.

Main function:

def get_connection(profile: str = "") -> psycopg.Connection:

The function uses psycopg to connect to PostgreSQL.

Connection details are resolved using environment variables through the common configuration utilities.

Source connection variables:

SRC_PG_HOST
SRC_PG_PORT
SRC_PG_DATABASE
SRC_PG_USER
SRC_PG_PASSWORD

The source PostgreSQL port defaults to 5432 if another value is not supplied.

The profile argument allows the connection configuration to use profile-specific environment values.

The connection is opened with autocommit disabled.

The connection object is returned to the shared Extract/Load framework.

Important distinction:

connections.py handles the SOURCE PostgreSQL database.

The destination PostgreSQL connection is handled separately through common/target.py.


3.2 extract.py – PostgreSQL Extraction

Location:
src/extract_load/postgres/extract.py

Purpose:
Contains the PostgreSQL-specific functions used to retrieve source data.

The main functions imported by run.py are:

extract_batches
count_rows

extract_batches:
Supplies source records to the shared loading framework in batches.

This allows the framework to process data without requiring all records to be loaded into memory at once.

count_rows:
Retrieves the source-side record count.

The count is used by the verification process to compare source and destination tables.

The full internal implementation of these functions was not captured in the review screenshots. Their SQL queries, parameters, and batching details should be checked directly before modifying extraction behavior.


3.3 run.py – PostgreSQL Execution Controller

Location:
src/extract_load/postgres/run.py

Purpose:
Connects PostgreSQL-specific functionality with the common Extract/Load framework.

The main function follows this structure:

def main() -> int:
    return run_el(
        source_name="PostgreSQL",
        default_config_path=DEFAULT_CONFIG_PATH,
        connect_source=get_connection,
        extract_batches=extract_batches,
        count_source=count_rows,
    )

The function identifies PostgreSQL as the source system.

It provides the following components to the shared framework:

- Source database connection
- Source data extraction
- Source row counting
- Default table configuration path

The default table configuration is:

src/extract_load/postgres/tables.yml

Most of the execution logic is delegated to common/cli.py.

This keeps the PostgreSQL-specific execution controller relatively small.


3.4 __main__.py – Python Entry Point

Location:
src/extract_load/postgres/__main__.py

Purpose:
Allows the PostgreSQL module to be executed directly through Python.

Implementation:

import sys

from extract_load.postgres.run import main

sys.exit(main())

When the PostgreSQL module is executed, Python calls the main() function in run.py.

The return value becomes the process exit code.

This is important for GitLab CI/CD because the pipeline uses the process exit status to identify successful and failed executions.


3.5 __init__.py – Package Initialization

Location:
src/extract_load/postgres/__init__.py

Purpose:
Identifies the PostgreSQL directory as a Python package.

The file appeared empty during the review.

The actual connection, extraction, and execution logic is located in the other PostgreSQL modules.


3.6 tables.yml – Table Mapping Configuration

Location:
src/extract_load/postgres/tables.yml

Purpose:
Defines the PostgreSQL source tables and their destination mappings.

Although this is a YAML file rather than a Python file, it is required by the PostgreSQL execution framework.

The configuration supports:

source_schema
source_table
target_schema
target_table
incremental
indexes
unlogged
tags

The configuration can define individual tables or groups of tables.

Grouped tables can inherit shared configuration values.

The Python framework reads these mappings at runtime.

This allows table definitions to be maintained separately from the Python execution code.

The full table list is maintained in the configuration file and was not individually validated during this review.


4. COMMON PYTHON MODULES
-------------------------

The PostgreSQL implementation uses shared functions located in:

src/extract_load/common/


4.1 cli.py – Main Extract/Load Framework

Location:
src/extract_load/common/cli.py

Purpose:
Controls the overall Extract/Load execution.

Main function:

run_el()

Responsibilities:

- Load environment configuration
- Parse command-line arguments
- Read table mappings
- Apply table selection
- Establish database connections
- Execute table loading
- Run optional verification
- Track individual table results
- Log failures
- Return process exit codes

Supported command-line options identified:

--config
--select
--exclude
--jobs
--full-refresh
--verify-counts
--verify-only
--verify-output

--config:
Specifies an alternative table configuration file.

--select:
Limits execution to selected tables or tags.

--exclude:
Excludes selected tables from execution.

--jobs:
Controls concurrent worker execution.

--full-refresh:
Requests a full refresh of applicable tables.

--verify-counts:
Runs source-versus-target count verification.

--verify-only:
Skips loading and performs count verification only.

--verify-output:
Writes verification results to a JSON file.

Verification-only behavior:

The code checks args.verify_only and skips the loading operation.

It still opens source and target database connections and performs count queries.

This makes it useful for validating existing source and target data without executing a new load.

Failure handling:

The framework tracks individual table failures.

If one or more tables fail, the Python process returns a nonzero exit code.

GitLab can detect that exit code, although the final pipeline status also depends on the job's allow_failure setting.


4.2 config.py – Table Configuration Processing

Location:
src/extract_load/common/config.py

Purpose:
Reads and processes Extract/Load table configurations.

Main function:

def load_mappings(config_path: Path) -> list[TableMapping]:

The function reads the YAML configuration file and builds the table mappings used during execution.

The configuration supports individual table entries and table groups.

Group-level settings can be inherited by individual tables.

The module also includes the resolve_env() utility used to retrieve environment variables, including profile-specific settings.

This allows the source connection modules to share the same environment-resolution approach.


4.3 env.py – Environment Configuration

Location:
src/extract_load/common/env.py

Purpose:
Loads Extract/Load environment settings and resolves target database configuration.

Important functions:

el_env_file()
load_el_env()
target_credentials()
target_endpoint()

Environment file resolution:

The implementation checks for EL_ENV_FILE first.

EL_ENV_FILE provides an explicit path to an environment configuration file.

If an explicit file is not configured, the code can fall back to:

extract_load.env

This fallback file is located in the repository root.

An explicit EL_ENV_FILE path that does not exist results in a configuration error.

Environment variables already defined in the process take precedence over values loaded from the environment file.

This is relevant for GitLab because CI/CD variables can be supplied directly.

Target credentials:

The implementation supports dedicated EL credentials:

TARGET_PG_USER
TARGET_PG_PASSWORD

If dedicated credentials are not configured, it can use:

DBT_PG_USER
DBT_PG_PASSWORD

The dedicated username and password must be configured together.

Target endpoint:

The target host, port, and database name are resolved from:

dbt/profiles.yml

The active environment is selected using DBT_TARGET.

This keeps the Extract/Load destination aligned with the dbt target configuration.


4.4 target.py – Destination PostgreSQL Operations

Location:
src/extract_load/common/target.py

Purpose:
Manages the destination PostgreSQL connection and shared target-side operations.

Main connection function:

get_pg_connection()

The target connection uses:

- The endpoint resolved from dbt/profiles.yml
- The credentials resolved through common/env.py
- psycopg for PostgreSQL connectivity

The connection is opened with autocommit disabled.

Dependent view handling:

The implementation contains:

_capture_dependent_views()
_recreate_dependent_views()

These functions help preserve views and materialized views that depend on a table being replaced.

Before a table replacement, dependent views can be identified and captured.

After the replacement, the views can be recreated.

The module also uses a threading lock for certain target DDL operations.

This helps prevent concurrent schema modifications from interfering with one another.


4.5 verify.py – Data Verification

Location:
src/extract_load/common/verify.py

Purpose:
Supports source-versus-target row-count verification.

The common execution framework uses the verification functions when verification is requested.

The reviewed implementation treats these conditions as failures:

- Target count is less than source count
- Target table is missing
- An error occurs during counting

A target count greater than the source count is reported as a warning rather than automatically treated as a failure.

This distinction matters for incremental loading because source deletions may not always be propagated to the target.

Verification results can be written to a JSON report.


4.6 selection.py – Table Selection

Location:
src/extract_load/common/selection.py

Purpose:
Provides shared functionality related to selecting tables for execution.

The overall framework supports selection by table name and tags.

The complete internal implementation of this file was not reviewed.

Its functions should be checked directly before modifying table-selection behavior.


4.7 state.py – State Management

Location:
src/extract_load/common/state.py

This file is present in the common package.

Its internal implementation was not reviewed in detail.

The specific state-management responsibilities need to be confirmed from the source code.


4.8 sync.py – Shared Synchronization Logic

Location:
src/extract_load/common/sync.py

This file is part of the common package.

Its internal implementation was not fully reviewed.

The synchronization functions and their relationship to table loading or incremental processing should be confirmed directly from the source code.


5. POSTGRESQL EXECUTION FLOW
-----------------------------

The overall PostgreSQL execution follows this structure:

Step 1:
Python starts the PostgreSQL module.

File:
postgres/__main__.py

Step 2:
The entry point calls the PostgreSQL execution controller.

File:
postgres/run.py

Step 3:
The execution controller calls the shared run_el() function.

File:
common/cli.py

Step 4:
The framework loads environment configuration.

File:
common/env.py

Step 5:
The framework reads table mappings.

Files:
common/config.py
postgres/tables.yml

Step 6:
The framework establishes the source PostgreSQL connection.

File:
postgres/connections.py

Step 7:
The framework establishes the target PostgreSQL connection.

File:
common/target.py

Step 8:
The PostgreSQL extraction module supplies source data.

File:
postgres/extract.py

Step 9:
The shared framework processes the selected table mappings and loads data into the destination.

Step 10:
Optional verification compares source and target row counts.

Step 11:
Execution results and failures are logged.

Step 12:
The Python process returns an exit code.

Overall flow:

postgres/__main__.py
        |
        v
postgres/run.py
        |
        v
common/cli.py
        |
        +--> common/env.py
        |
        +--> common/config.py
        |         |
        |         v
        |    postgres/tables.yml
        |
        +--> postgres/connections.py
        |         |
        |         v
        |    Source PostgreSQL
        |
        +--> common/target.py
        |         |
        |         v
        |    Target PostgreSQL
        |
        +--> postgres/extract.py
        |
        +--> Shared loading logic
        |
        +--> Optional verification
        |
        v
Logs and process exit status


6. POSTGRESQL ENVIRONMENT VARIABLES
------------------------------------

Source PostgreSQL variables:

SRC_PG_HOST
SRC_PG_PORT
SRC_PG_DATABASE
SRC_PG_USER
SRC_PG_PASSWORD

Destination PostgreSQL variables:

DBT_TARGET
DBT_PG_USER
DBT_PG_PASSWORD

Optional dedicated target credentials:

TARGET_PG_USER
TARGET_PG_PASSWORD

Environment configuration:

EL_ENV_FILE

GitLab execution controls:

EL_VERIFY_ONLY
DBT_SELECT
DBT_LOAD_MODE

These variables serve different purposes.

The SRC_PG variables are used to connect to the upstream PostgreSQL database.

The DBT_PG or TARGET_PG credentials are used for the destination PostgreSQL database.

DBT_TARGET determines the destination environment.

EL_ENV_FILE identifies the shared Extract/Load environment configuration.

EL_VERIFY_ONLY, DBT_SELECT, and DBT_LOAD_MODE are used by the GitLab pipeline to control execution behavior.

Database credentials should remain outside the committed source code.


7. RUNNING THE POSTGRESQL PYTHON MODULE
----------------------------------------

The project uses uv for Python dependency management and command execution.

The Python version specified in pyproject.toml is:

requires-python = "==3.14.3"

The following commands are relevant to the PostgreSQL implementation.

Check the PostgreSQL CLI:

uv run --no-sync extract-load-postgres --help

Execute the PostgreSQL Python module:

uv run --no-sync python -m extract_load.postgres

The second command may execute actual data loading, depending on the environment and arguments.

It should only be used after confirming the destination and table selection.

Run verification for a selected configured table:

uv run --no-sync extract-load-postgres --select example_reference_table --verify-only

Replace example_reference_table with a valid configured target table name.

The --verify-only option skips the load but still requires database connectivity.

The --no-sync option prevents uv from synchronizing dependencies before execution.

It does not install missing packages.


8. GITLAB CI/CD INTEGRATION
----------------------------

The GitLab pipeline is configured in:

.gitlab-ci.yml

The PostgreSQL Extract/Load job is named:

postgres_extract_load

The job extends the shared upstream Extract/Load configuration:

.el_upstream

The PostgreSQL-specific configuration includes:

postgres_extract_load:
  extends: .el_upstream
  variables:
    EL_MODULE: "postgres"
    EL_LABEL: "PostgreSQL Extract/Load"

The shared .el_job template contains the Python execution logic.

The pipeline constructs the Python command using EL_MODULE.

For PostgreSQL, the module argument becomes:

extract_load.postgres

The shared job also supports:

EL_VERIFY_ONLY
DBT_LOAD_MODE
DBT_SELECT
DBT_EXCLUDE

When EL_VERIFY_ONLY is enabled, the job passes the verification-only option to Python.

When DBT_LOAD_MODE is full, the job can pass --full-refresh.

When DBT_SELECT is populated, the job passes the selected tables to the Python command.

The shared job generates log artifacts.

The relevant artifact locations include:

logs/<CI_JOB_NAME>.log

For verification-only runs, a JSON report can also be generated:

verify/<CI_JOB_NAME>.json

The reviewed GitLab configuration retains these artifacts for 30 days.

The Extract/Load jobs currently have:

allow_failure: true

This means the GitLab pipeline may continue even when an Extract/Load job fails.

The Python process itself can still return a nonzero exit code.

The job's GitLab status and the overall pipeline status should therefore be reviewed separately.

The shared upstream Extract/Load rules currently target the default branch for production extraction.

The lower environments are intended to receive production raw data through the mirroring process.

Feature-branch testing requires an appropriate isolated test rule or an approved alternative execution path.


9. PRODUCTION EXTRACTION AND LOWER-ENVIRONMENT MIRRORING
--------------------------------------------------------

The repository documentation describes a production-only upstream Extract/Load approach.

Under this approach:

Production:
Runs extraction from the upstream source systems into the production Business Layer PostgreSQL database.

Development and Test:
Receive copies of production raw landing data through the mirroring process.

The mirror implementation is located under:

src/extract_load/mirror/

The mirror job in GitLab is named:

prod_mirror

Its purpose is to populate lower environments from production raw data rather than requiring development and test to connect independently to every upstream source.

The mirroring process uses the configured production database access and environment-specific destination settings.

This separation is important when validating the production execution design.


10. VALIDATION COMPLETED
-------------------------

The following checks were performed on the PostgreSQL Extract/Load implementation.

Working branch:

feature/postgres-el-cicd-testing

Git working tree:
Clean during the offline validation.

Python version:
3.14.3

uv version:
0.11.29

Python syntax validation:

python -m compileall -q src/extract_load

Result:
Passed.

Git whitespace validation:

git diff --check

Result:
No whitespace errors reported.

PostgreSQL module inspection:
Completed.

Files confirmed:

__init__.py
__main__.py
connections.py
extract.py
run.py
tables.yml

Environment configuration review:
