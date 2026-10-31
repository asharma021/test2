Set-Location C:\Users\SharmaAashishR\oa-business-layer
$ErrorActionPreference = 'Stop'
if ((git branch --show-current) -ne 'pipeline-testing') { throw 'STOP: not on pipeline-testing' }
git fetch origin --prune 2>&1 | Out-Null
if ((git rev-parse HEAD) -ne (git rev-parse origin/pipeline-testing)) { throw 'STOP: HEAD differs from origin/pipeline-testing' }
if (git status --porcelain -- .gitlab-ci.yml) { throw 'STOP: .gitlab-ci.yml already has uncommitted changes' }
$want = '\\thqpcelerra01\ttb_common\_OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'
"Folder visible from this machine: $(Test-Path -LiteralPath (Split-Path $want))   File: $(Test-Path -LiteralPath $want)"
$p = Join-Path $PWD '.gitlab-ci.yml'
$b = [IO.File]::ReadAllBytes($p); $bom = $b.Length -ge 3 -and $b[0] -eq 0xEF -and $b[1] -eq 0xBB
$raw = [IO.File]::ReadAllText($p); $nl = if ($raw.Contains("`r`n")) { "`r`n" } else { "`n" }
$L = $raw -split "`r?`n"
$s = [array]::IndexOf($L, 'postgres_extract_load_dev_incremental:'); if ($s -lt 0) { throw 'STOP: incremental job not found' }
$hit = -1; for ($k = $s + 1; $k -lt $L.Count -and $L[$k] -notmatch '^\S'; $k++) { if ($L[$k] -match '^\s+EL_ENV_FILE:') { $hit = $k; break } }
if ($hit -lt 0) { throw 'STOP: EL_ENV_FILE not found in incremental job' }
"Before: $($L[$hit].Trim())"
$L[$hit] = "    EL_ENV_FILE: '$want'"
"After : $($L[$hit].Trim())"
[IO.File]::WriteAllText($p, ($L -join $nl), (New-Object Text.UTF8Encoding($bom)))
$ErrorActionPreference = 'Continue'
$add = @(git diff -U0 -- .gitlab-ci.yml | Where-Object { $_ -match '^\+[^+]' }).Count
$del = @(git diff -U0 -- .gitlab-ci.yml | Where-Object { $_ -match '^-[^-]' }).Count
git diff -- .gitlab-ci.yml
"Changed lines: +$add -$del (expect +1 -1)"
if ($add -eq 1 -and $del -eq 1) {
  git add -- .gitlab-ci.yml
  git commit -q -m "Fix DEV EL_ENV_FILE path (_OA_Data_Sharing) for DEV incremental/verify jobs" -m "Runner reported the previous path does not exist. Incremental job stays disabled; postgres_extract_load_dev_verify inherits this value."
  git push -o ci.skip origin pipeline-testing
  git log --oneline -2
  'RESULT: FIXED, COMMITTED AND PUSHED (no pipeline started)'
} elseif ($add -eq 0 -and $del -eq 0) {
  'RESULT: path already correct in the file - nothing changed. Send me this output.'
} else {
  git checkout -- .gitlab-ci.yml
  'RESULT: unexpected change - reverted, nothing committed. Send me this output.'
}
