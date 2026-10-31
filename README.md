Set-Location C:\Users\SharmaAashishR\oa-business-layer
$ErrorActionPreference = 'Stop'
if ((git branch --show-current) -ne 'pipeline-testing') { throw 'STOP: not on pipeline-testing' }
git fetch origin --prune 2>&1 | Out-Null
if ((git rev-parse HEAD) -ne (git rev-parse origin/pipeline-testing)) { throw 'STOP: HEAD differs from origin/pipeline-testing' }
if (git status --porcelain -- .gitlab-ci.yml) { throw 'STOP: .gitlab-ci.yml already has uncommitted changes' }
$cfg = 'src/extract_load/postgres/tables_ivv_verify.yml'
if (-not (Test-Path -LiteralPath $cfg -PathType Leaf)) { throw "STOP: $cfg not found" }
$p = Join-Path $PWD '.gitlab-ci.yml'
$b = [IO.File]::ReadAllBytes($p); $bom = $b.Length -ge 3 -and $b[0] -eq 0xEF -and $b[1] -eq 0xBB
$raw = [IO.File]::ReadAllText($p); $nl = if ($raw.Contains("`r`n")) { "`r`n" } else { "`n" }
$L = [Collections.Generic.List[string]]($raw -split "`r?`n")
$s = $L.IndexOf('postgres_extract_load_ivv_incremental:'); if ($s -lt 0) { throw 'STOP: IVV incremental job not found' }
$lab = -1; $r = -1
for ($k = $s + 1; $k -lt $L.Count -and $L[$k] -notmatch '^\S'; $k++) {
  if ($L[$k] -match '^\s+EL_CONFIG:|^\s+DBT_SELECT:|^\s+EL_ENV_FILE:') { throw 'STOP: IVV job already has EL_CONFIG/DBT_SELECT/EL_ENV_FILE' }
  if ($lab -lt 0 -and $L[$k] -match '^\s+EL_LABEL:') { $lab = $k }
  if ($L[$k] -eq '  rules:') { $r = $k }
}
if ($lab -lt 0) { throw 'STOP: EL_LABEL not found in IVV job' }
if ($r -lt 0 -or $L[$r + 1] -ne '    - when: never') { throw 'STOP: expected "rules: - when: never" not found in IVV job' }
$L.InsertRange($r + 1, [string[]]@("    - if: '`$CI_COMMIT_BRANCH == `"pipeline-testing`" && `$CI_PIPELINE_SOURCE == `"web`"'", '      when: manual'))
$L.InsertRange($lab + 1, [string[]]@("    EL_CONFIG: `"$cfg`"", "    EL_ENV_FILE: '\\thqpcelerra01\ttb_common\_OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'", '    DBT_SELECT: "myttb_cg_ref_identifier_type"'))
[IO.File]::WriteAllText($p, ($L -join $nl), (New-Object Text.UTF8Encoding($bom)))
$ErrorActionPreference = 'Continue'
git diff -- .gitlab-ci.yml
$add = @(git diff -U0 -- .gitlab-ci.yml | Where-Object { $_ -match '^\+[^+]' }).Count
$del = @(git diff -U0 -- .gitlab-ci.yml | Where-Object { $_ -match '^-[^-]' }).Count
"Changed lines: +$add -$del (expect +5 -0)"
if ($add -eq 5 -and $del -eq 0) {
  git add -- .gitlab-ci.yml
  git commit -q -m "Configure and enable manual IVV load job (IVV source -> IVV target, single table)" -m "postgres_extract_load_ivv_incremental: tables_ivv_verify.yml (source profile IVV), myttb_cg_ref_identifier_type, extract_load.env; manual on pipeline-testing web only. IVV target inspected read-only first: raw_myttb schema and table absent."
  git push -o ci.skip origin pipeline-testing
  git log --oneline -2
  'RESULT: IVV JOB CONFIGURED (manual only), COMMITTED AND PUSHED (no pipeline started)'
} else {
  git checkout -- .gitlab-ci.yml
  'RESULT: unexpected change - reverted, nothing committed. Send me this output.'
}
