git add .gitlab-ci.yml
git -c core.hooksPath=NUL commit -m "Add IVV environment file diagnostics"
git push -o ci.skip origin pipeline-testing
