Set-Location 'C:\Users\SharmaAashishR\oa-business-layer'
git checkout el-env-file-upstream-jobs
$ci = Get-Content '.gitlab-ci.yml'

function Show-Block([string]$job) {
    $start = -1
    for ($i = 0; $i -lt $ci.Count; $i++) { if ($ci[$i] -match "^$([regex]::Escape($job)):\s*$") { $start = $i; break } }
    if ($start -lt 0) { Write-Host "`n--- $job NOT FOUND ---" -ForegroundColor Red; return }
    Write-Host "`n--- $job ---" -ForegroundColor Cyan
    for ($j = $start; $j -lt $ci.Count; $j++) {
        if ($j -gt $start -and $ci[$j] -match '^\S') { break }
        "{0,5}: {1}" -f ($j + 1), $ci[$j]
    }
}

Show-Block '.el_job'
Show-Block '.el_upstream'
Show-Block 'oracle_extract_load'
Show-Block 'postgres_extract_load'

Write-Host "`n--- verify-only wiring ---" -ForegroundColor Cyan
for ($i = 0; $i -lt $ci.Count; $i++) {
    if ($ci[$i] -match 'verify_only|EL_VERIFY_ONLY|verify-only') { "{0,5}: {1}" -f ($i + 1), $ci[$i] }
}
