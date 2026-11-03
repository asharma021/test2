& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
$ci = @(git show origin/development:.gitlab-ci.yml | ForEach-Object { $_.TrimEnd("`r") })
"===== Lines containing 'workflow' ====="
for ($i = 0; $i -lt $ci.Count; $i++) { if ($ci[$i] -match 'workflow') { "{0,5}: [{1}]" -f ($i + 1), $ci[$i] } }
"===== Lines that are exactly '---' (YAML document separators) ====="
for ($i = 0; $i -lt $ci.Count; $i++) { if ($ci[$i] -eq '---') { "{0,5}: ---" -f ($i + 1) } }
"===== The workflow: block (25 lines) ====="
$w = [array]::IndexOf($ci, 'workflow:')
if ($w -ge 0) { for ($i = $w; $i -lt [Math]::Min($w + 25, $ci.Count); $i++) { "{0,5}: [{1}]" -f ($i + 1), $ci[$i] } } else { "(no line that is exactly 'workflow:')" }
}
