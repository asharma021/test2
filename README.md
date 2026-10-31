Set-Location C:\Users\SharmaAashishR\oa-business-layer
$ErrorActionPreference = 'Stop'
if ((git branch --show-current) -ne 'pipeline-testing') { throw 'STOP: not on pipeline-testing' }
git fetch origin --prune 2>&1 | Out-Null
if ((git rev-parse HEAD) -ne (git rev-parse origin/pipeline-testing)) { throw 'STOP: HEAD differs from origin/pipeline-testing' }
if (git status --porcelain -- .gitlab-ci.yml) { throw 'STOP: .gitlab-ci.yml already has uncommitted changes' }
$p = Join-Path $PWD '.gitlab-ci.yml'
$b = [IO.File]::ReadAllBytes($p); $bom = $b.Length -ge 3 -and $b[0] -eq 0xEF -and $b[1] -eq 0xBB
$raw = [IO.File]::ReadAllText($p); $nl = if ($raw.Contains("`r`n")) { "`r`n" } else { "`n" }
$L = [Collections.Generic.List[string]]($raw -split "`r?`n")
$s = $L.IndexOf('postgres_extract_load_dev_incremental:'); if ($s -lt 0) { throw 'STOP: incremental job not found' }
$r = -1; for ($k = $s + 1; $k -lt $L.Count -and $L[$k] -notmatch '^\S'; $k++) { if ($L[$k] -eq '  rules:') { $r = $k; break } }
if ($r -lt 0 -or $L[$r + 1] -ne '    - when: never') { throw 'STOP: expected "rules: - when: never" not found in incremental job' }
$L.InsertRange($r + 1, [string[]]@("    - if: '`$CI_COMMIT_BRANCH == `"pipeline-testing`" && `$CI_PIPELINE_SOURCE == `"web`"'", '      when: manual'))
[IO.File]::WriteAllText($p, ($L -join $nl), (New-Object Text.UTF8Encoding($bom)))
$ErrorActionPreference = 'Continue'
git diff -- .gitlab-ci.yml
$add = @(git diff -U0 -- .gitlab-ci.yml | Where-Object { $_ -match '^\+[^+]' }).Count
$del = @(git diff -U0 -- .gitlab-ci.yml | Where-Object { $_ -match '^-[^-]' }).Count
"Changed lines: +$add -$del (expect +2 -0)"
if ($add -eq 2 -and $del -eq 0) {
  git add -- .gitlab-ci.yml
  git commit -q -m "Enable manual DEV incremental load on pipeline-testing (myttb_controlled_group)" -m "Manual, web pipelines on protected pipeline-testing only; incremental guard unchanged. Pre-run inspection: 2,982 rows, 0 duplicate ids, existing watermark on last_modified_date."
  git push -o ci.skip origin pipeline-testing
  git log --oneline -2
  'RESULT: ENABLED (manual only), COMMITTED AND PUSHED (no pipeline started)'
} else {
  git checkout -- .gitlab-ci.yml
  'RESULT: unexpected change - reverted, nothing committed. Send me this output.'
}
