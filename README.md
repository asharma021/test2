& {
Set-Location 'C:\Users\SharmaAashishR\oa-business-layer'
git fetch origin production 2>$null | Out-Null
Write-Host '--- ITDS in production E&L config / CI (uncommented lines) ---' -ForegroundColor Cyan
git --no-pager grep -n -i -I itds origin/production -- 'src/extract_load/*' '.gitlab-ci.yml' |
  ForEach-Object { $_ -replace '^origin/production:','' } |
  Where-Object { $_ -notmatch ':\d+:\s*#' } | Write-Host
Write-Host '--- ITDS lines that are commented out ---' -ForegroundColor Yellow
git --no-pager grep -n -i -I itds origin/production -- 'src/extract_load/*' '.gitlab-ci.yml' |
  ForEach-Object { $_ -replace '^origin/production:','' } |
  Where-Object { $_ -match ':\d+:\s*#' } | Write-Host
}
