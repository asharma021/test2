
# Confirm GitLab CLI is installed
glab --version

# Confirm authentication
glab auth status

# Create Merge Request
$description = @"
## Summary
Configured the PostgreSQL Extract/Load GitLab CI/CD pipeline for DEV verification.

## Changes
- Added PostgreSQL EL job configuration and branch-specific rules.
- Configured approved shared environment file access.
- Enabled manual DEV verify-only execution.
- Moved state-table initialization after verify-only processing.
- Removed temporary diagnostic jobs.

## Validation
- GitLab pipeline #68171 completed successfully.
- PostgreSQL source and target counts matched (12/12).
- Verified 1 table with 0 failures.
- DEV verification completed without loading data.

## Next Steps
- Review and merge into development.
- Coordinate TEST environment configuration and validation.
- Promote through the standard approval process.
"@

glab mr create `
  --source-branch pipeline-testing `
  --target-branch development `
  --title "Configure PostgreSQL EL Verify-Only Pipeline for DEV" `
  --description $description
