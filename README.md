& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
git merge-base --is-ancestor origin/production origin/development
"1) production is now inside development: $($LASTEXITCODE -eq 0)"
git merge-tree --write-tree --name-only origin/production origin/development 2>&1 | Out-Null
"2) development -> production merges cleanly: $($LASTEXITCODE -eq 0)"
"3) development tip: $(git log -1 --format='%h %s' origin/development)"
}
