& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
"1) development tip : $(git log -1 --format='%h %an : %s' origin/development)"
$fix = [bool](git show origin/development:dbt/tests/generic/no_deprecated_abbreviations.sql | Select-String -SimpleMatch "flags.WHICH == 'compile'" -Quiet)
"2) fix on development: $fix"
"3) last change to test file: $(git log -1 --format='%h %an : %s' origin/development -- dbt/tests/generic/no_deprecated_abbreviations.sql)"
}
