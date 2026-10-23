Configure PostgreSQL EL Verify-Only Pipeline for DEV



Configured PostgreSQL Extract/Load CI/CD for DEV with branch-specific rules, shared environment file access, and manual verify-only execution.

Updated state-table initialization to avoid writes during verification and removed temporary diagnostics.

Validation: Pipeline #68171 passed. Source and target counts matched (12/12), with zero failures and no data loaded.

Next: Merge into development after review and proceed with TEST validation.
