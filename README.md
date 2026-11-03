& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
$rx = 'Claude|Anthropic|Co-Authored-By|Generated with|claude\.ai|claude\.com|written by AI|AI-generated'
"===== 1. Commit messages on any remote branch that mention it ====="
$seen = @{}
foreach ($b in (git branch -r --format='%(refname:short)' | Where-Object { $_ -notmatch 'HEAD' })) {
  git log --format='%h|%an|%s' -i -E --grep=$rx $b 2>$null | ForEach-Object { if (-not $seen[$_]) { $seen[$_] = 1; "   $_   (on $b)" } }
}
if (-not $seen.Count) { "   none" }
"===== 2. Files on development that mention it ====="
$f = git grep -n -i -E $rx origin/development -- . 2>$null
if ($f) { $f } else { "   none" }
}
