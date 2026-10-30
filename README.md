Continued PostgreSQL Extract/Load CI/CD implementation for DEV and IVV.

Added dedicated DEV and IVV incremental-load jobs with protected-branch restrictions, manual execution controls, environment validation, and safety prechecks.

Updated the configuration paths and verified the GitLab pipeline configuration. The existing IVV single-table verify-only test completed successfully with matching source and target row counts (12/12).

Incremental-load execution is currently pending confirmation that the required database tables and data are ready.

Next steps: Execute DEV incremental-load testing once the data is available, validate the loaded records and checkpointing behavior, then proceed with IVV incremental-load testing and post-load verification.
