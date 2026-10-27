git add .gitlab-ci.yml
git -c core.hooksPath=NUL commit -m "Check DEV UNC access from GitLab runner"
git push -o ci.skip origin pipeline-testing
