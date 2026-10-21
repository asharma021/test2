git diff --check
git diff -- .gitlab-ci.yml


git add .gitlab-ci.yml
git commit -m "Diagnose EL shared directory access"
git push origin pipeline-testing
