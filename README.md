
git push origin pipeline-testing `
  -o merge_request.create `
  -o merge_request.target=development `
  -o "merge_request.title=Configure PostgreSQL EL Verify-Only Pipeline for DEV" `
  -o "merge_request.description=Configured PostgreSQL EL DEV verify-only pipeline with shared environment settings and branch rules. Validation passed in pipeline 68171 with matching source and target counts (12/12), one table verified, and zero failures. Ready for review and TEST preparation."
