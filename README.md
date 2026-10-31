Set-Location C:\Users\SharmaAashishR\oa-business-layer
$ErrorActionPreference = 'Stop'
if ((git branch --show-current) -ne 'pipeline-testing') { throw 'STOP: not on pipeline-testing' }
git fetch origin --prune 2>&1 | Out-Null
if ((git rev-parse HEAD) -ne (git rev-parse origin/pipeline-testing)) { throw 'STOP: HEAD differs from origin/pipeline-testing' }
if (git status --porcelain -- .gitlab-ci.yml) { throw 'STOP: .gitlab-ci.yml already has uncommitted changes' }
$want = 'src/extract_load/postgres/tables_ivv_verify.yml'
if (-not (Test-Path -LiteralPath $want -PathType Leaf)) { throw "STOP: $want not found locally" }
$p = Join-Path $PWD '.gitlab-ci.yml'
$b = [IO.File]::ReadAllBytes($p); $bom = $b.Length -ge 3 -and $b[0] -eq 0xEF -and $b[1] -eq 0xBB
$raw = [IO.File]::ReadAllText($p); $nl = if ($raw.Contains("`r`n")) { "`r`n" } else { "`n" }
$L = $raw -split "`r?`n"
$s = [array]::IndexOf($L, 'postgres_extract_load_test:'); if ($s -lt 0) { throw 'STOP: postgres_extract_load_test not found' }
$hit = -1; for ($k = $s + 1; $k -lt $L.Count -and $L[$k] -notmatch '^\S'; $k++) { if ($L[$k] -match '^\s+EL_CONFIG:') { $hit = $k; break } }
if ($hit -lt 0) { throw 'STOP: EL_CONFIG not found in postgres_extract_load_test' }
$old = ($L[$hit] -replace '^\s*EL_CONFIG:\s*["'']?', '') -replace '["'']\s*$', ''
"Before: [$old]  length $($old.Length)  exists: $(Test-Path -LiteralPath $old)"
$L[$hit] = "    EL_CONFIG: `"$want`""
"After : [$want]  length $($want.Length)  exists: $(Test-Path -LiteralPath $want)"
[IO.File]::WriteAllText($p, ($L -join $nl), (New-Object Text.UTF8Encoding($bom)))
$ErrorActionPreference = 'Continue'
git diff -- .gitlab-ci.yml
$add = @(git diff -U0 -- .gitlab-ci.yml | Where-Object { $_ -match '^\+[^+]' }).Count
$del = @(git diff -U0 -- .gitlab-ci.yml | Where-Object { $_ -match '^-[^-]' }).Count
"Changed lines: +$add -$del (expect +1 -1)"
if ($add -eq 1 -and $del -eq 1 -and $want.Length -eq 47) {
  git add -- .gitlab-ci.yml
  git commit -q -m "Fix EL_CONFIG path typo in IVV verify job" -m "postgres_extract_load_test EL_CONFIG was one character short of src/extract_load/postgres/tables_ivv_verify.yml (runner: Config file not found). Job remains manual and verify-only."
  git push -o ci.skip origin pipeline-testing
  git log --oneline -2
  'RESULT: FIXED, COMMITTED AND PUSHED (no pipeline started)'
} elseif ($add -eq 0 -and $del -eq 0) {
  'RESULT: no change made - send me this output.'
} else {
  git checkout -- .gitlab-ci.yml
  'RESULT: unexpected change - reverted, nothing committed. Send me this output.'
}
