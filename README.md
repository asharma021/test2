
git status --short
git branch --show-current

git push origin pipeline-testing `
  -o merge_request.create `
  -o merge_request.target=development `
  -o "merge_request.title=Configure PostgreSQL EL Verify-Only Pipeline for DEV" `
  -o "merge_request.description=$description"
