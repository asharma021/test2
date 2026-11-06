& {
    $ErrorActionPreference = 'Continue'
    Set-Location 'C:\Users\SharmaAashishR\oa-business-layer'
    git checkout -- .gitlab-ci.yml
    $dirty = git status --porcelain | Where-Object { $_ -notmatch '\.gitlab-ci\.yml\.backup$' }
    if ($dirty) { Write-Host "STOP: uncommitted changes:`n$dirty" -ForegroundColor Red; return }
    git checkout el-env-file-upstream-jobs
    if ($LASTEXITCODE -ne 0) { Write-Host "STOP: could not switch branch" -ForegroundColor Red; return }

    $path = (Resolve-Path '.gitlab-ci.yml').Path
    $text = [System.IO.File]::ReadAllText($path)
    if ($text -match 'TEMP: dev verify-only check') { Write-Host "STOP: temp rule already in file" -ForegroundColor Yellow; return }
    $nl = if ($text -match "`r`n") { "`r`n" } else { "`n" }
    $lines = [System.Collections.Generic.List[string]]::new([string[]]($text -split "`r?`n"))

    $start = -1
    for ($i = 0; $i -lt $lines.Count; $i++) { if ($lines[$i] -match '^\.el_upstream:\s*$') { $start = $i; break } }
    if ($start -lt 0) { Write-Host "STOP: .el_upstream not found" -ForegroundColor Red; return }
    $end = $lines.Count
    for ($j = $start + 1; $j -lt $lines.Count; $j++) { if ($lines[$j] -match '^\S') { $end = $j; break } }

    $indent = $null
    for ($j = $start; $j -lt $end; $j++) {
        if ($lines[$j] -match '^(\s*)-\s*if:\s*[''"]?\$CI_COMMIT_BRANCH\s*==\s*\$CI_DEFAULT_BRANCH') { $indent = $Matches[1]; $prodLine = $j + 1 }
    }
    if ($null -eq $indent -or $indent.Length -eq 0) { Write-Host "STOP: production rule not found in .el_upstream" -ForegroundColor Red; return }
    Write-Host "Production rule found on line $prodLine (indent $($indent.Length) spaces)" -ForegroundColor Green

    $last = $end - 1
    while ($last -gt $start -and [string]::IsNullOrWhiteSpace($lines[$last])) { $last-- }

    $new = [string[]]@(
        ($indent + '# TEMP: dev verify-only check of upstream source settings. Remove before promoting to test.'),
        ($indent + '- if: ''$CI_COMMIT_BRANCH == "development" && $CI_PIPELINE_SOURCE == "web"'''),
        ($indent + '  variables:'),
        ($indent + '    DBT_TARGET: "dev"'),
        ($indent + '    EL_VERIFY_ONLY: "true"'),
        ($indent + '  when: manual')
    )
    if ($new.Count -ne 6) { Write-Host "STOP: expected 6 lines, got $($new.Count)" -ForegroundColor Red; return }
    $lines.InsertRange($last + 1, $new)
    [System.IO.File]::WriteAllText($path, ($lines -join $nl), (New-Object System.Text.UTF8Encoding($false)))

    Write-Host "`n=== numstat ===" -ForegroundColor Cyan
    git diff --numstat
    Write-Host "`n=== diff ===" -ForegroundColor Cyan
    git diff
    Write-Host "`nNOT COMMITTED. Send the output. To undo: git checkout -- .gitlab-ci.yml" -ForegroundColor Yellow
}
