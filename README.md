& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
"1) remote-dev fix branch on GitLab : $([bool](git ls-remote --heads origin fix/abbrev-test-compile-remote-dev))"
"2) its latest commit               : $(git log -1 --format='%h %s' origin/fix/abbrev-test-compile-remote-dev 2>$null)"
"3) MR !6 branch latest commit      : $(git log -1 --format='%h %s' origin/feature/prod-el-readiness)"
"4) current local branch            : $(git rev-parse --abbrev-ref HEAD)"
}
