& {
    $ErrorActionPreference = 'Continue'
    Set-Location 'C:\Users\SharmaAashishR\oa-business-layer'
    $dirty = git status --porcelain | Where-Object { $_ -notmatch '\.gitlab-ci\.yml\.backup$' }
    if ($dirty) { Write-Host "STOP: uncommitted changes:`n$dirty" -ForegroundColor Red; return }
    git checkout el-env-file-upstream-jobs
    if ($LASTEXITCODE -ne 0) { Write-Host "STOP: could not switch branch" -ForegroundColor Red; return }

    $path = (Resolve-Path '.gitlab-ci.yml').Path
    $text = [System.IO.File]::ReadAllText($path)
    if ($text -match 'TEMP: dev verify-only check') { Write-Host "STOP: temp rule already in file" -ForegroundColor Yellow; return }
    $nl = if ($text -match "`r`n") { "`r`n" } else { "`n" }
    Write-Host ("Line endings: " + $(if ($nl -eq "`r`n") {'CRLF'} else {'LF'}))
    $lines = [System.Collections.Generic.List[string]]::new([string[]]($text -split "`r?`n"))

    $start = -1
    for ($i = 0; $i -lt $lines.Count; $i++) { if ($lines[$i] -match '^\.el_upstream:\s*$') { $start = $i; break } }
    if ($start -lt 0) { Write-Host "STOP: .el_upstream not found" -ForegroundColor Red; return }
    $end = $lines.Count
    for ($j = $start + 1; $j -lt $lines.Count; $j++) { if ($lines[$j] -match '^\S') { $end = $j; break } }

    $indent = $null
    for ($i = $start; $i -lt $end; $i++) {
        if ($lines[$i] -match '^(\s*)- if: ''\$CI_COMMIT_BRANCH == \$CI_DEFAULT_BRANCH''') { $indent = $Matches[1] }
    }
    if ($null -eq $indent) { Write-Host "STOP: production rule not found in .el_upstream" -ForegroundColor Red; return }
    $last = $end - 1
    while ($last -gt $start -and [string]::IsNullOrWhiteSpace($lines[$last])) { $last-- }

    $new = @(
        $indent + '# TEMP: dev verify-only check of upstream source settings. Remove before promoting to test.',
        $indent + '- if: ''$CI_COMMIT_BRANCH == "development" && $CI_PIPELINE_SOURCE == "web"''',
        $indent + '  variables:',
        $indent + '    DBT_TARGET: "dev"',
        $indent + '    EL_VERIFY_ONLY: "true"',
        $indent + '  when: manual'
    )
    $lines.InsertRange($last + 1, [string[]]$new)
    [System.IO.File]::WriteAllText($path, ($lines -join $nl), (New-Object System.Text.UTF8Encoding($false)))

    Write-Host "`n=== numstat ===" -ForegroundColor Cyan
    git diff --numstat
    Write-Host "`n=== diff ===" -ForegroundColor Cyan
    git diff
    Write-Host "`nNOT COMMITTED. Send the output. To undo: git checkout -- .gitlab-ci.yml" -ForegroundColor Yellow
}
