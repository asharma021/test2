# Show total number of lines
(Get-Content .gitlab-ci.yml).Count

# Display the final diagnostic jobs
Get-Content .gitlab-ci.yml | Select-Object -Skip 1405

# Confirm branch and working tree
git status --short --branch
