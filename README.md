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
if ($L.Contains('postgres_extract_load_dev_verify:')) { throw 'STOP: dev_verify job already exists' }
$i = $L.IndexOf('postgres_extract_load_dev_incremental:'); if ($i -lt 0) { throw 'STOP: incremental job not found' }
$lab = -1; for ($k = $i + 1; $k -lt $L.Count -and $L[$k] -notmatch '^\S'; $k++) { if ($L[$k] -match '^\s+EL_LABEL:') { $lab = $k; break } }
if ($lab -lt 0) { throw 'STOP: EL_LABEL not found in incremental job' }
$L.InsertRange($lab + 1, [string[]]@('    EL_CONFIG: "src/extract_load/postgres/tables.yml"', "    EL_ENV_FILE: '\\thqpcelerra01\ttb_common\OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'", '    DBT_SELECT: "myttb_controlled_group"'))
$t = $L.IndexOf('postgres_extract_load_test:'); if ($t -lt 0) { throw 'STOP: postgres_extract_load_test not found' }
$L.InsertRange($t, [string[]]@('postgres_extract_load_dev_verify:', '  extends: postgres_extract_load_dev_incremental', '  variables:', '    EL_LABEL: "PostgreSQL DEV Verify-Only"', '    EL_VERIFY_ONLY: "true"', '  before_script:', '    - uv sync --quiet', '  rules:', "    - if: '`$CI_COMMIT_BRANCH == `"pipeline-testing`" && `$CI_PIPELINE_SOURCE == `"web`"'", '      when: manual', ''))
[IO.File]::WriteAllText($p, ($L -join $nl), (New-Object Text.UTF8Encoding($bom)))
$ErrorActionPreference = 'Continue'
'========== RESULT =========='
git diff --stat
git diff -- .gitlab-ci.yml
$del = @(git diff -U0 -- .gitlab-ci.yml | Where-Object { $_ -match '^-[^-]' }).Count
$add = @(git diff -U0 -- .gitlab-ci.yml | Where-Object { $_ -match '^\+[^+]' }).Count
$n = [IO.File]::ReadAllLines($p); $s = [array]::IndexOf($n, 'postgres_extract_load_dev_incremental:')
$e = $s + 1; while ($e -lt $n.Count -and $n[$e] -notmatch '^\S') { $e++ }
$neverOk = ($n[$s..($e - 1)] -join "`n") -match 'rules:\s*\r?\n\s*- when: never'
"Added lines: $add (expect 13)   Deleted lines: $del (expect 0)   Incremental job still disabled: $neverOk"
if ($add -eq 13 -and $del -eq 0 -and $neverOk) { 'LOCAL CHECK: PASS - now do the GitLab Validate step' } else { 'LOCAL CHECK: FAIL - undo with: git checkout -- .gitlab-ci.yml' }
Get-Content $p -Raw | Set-Clipboard
'File copied to clipboard for GitLab Pipeline editor > Validate.'
