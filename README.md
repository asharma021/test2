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
if ($L.Contains('prod_mirror_dev_verify:')) { throw 'STOP: prod_mirror_dev_verify already exists' }
$m = $L.IndexOf('prod_mirror:'); if ($m -lt 0) { throw 'STOP: prod_mirror not found' }
$e = $m + 1; while ($e -lt $L.Count -and ($L[$e] -match '^\s' -or $L[$e] -eq '')) { $e++ }
$job = @'
# READ-ONLY mirror readiness check on pipeline-testing: confirms the MIRROR_PG_*
# settings exist (names and SET/EMPTY only, never values), then runs prod_mirror
# in --verify-only mode (row counts prod raw vs DEV raw, no writes) for one table.
# Separate job: the approved prod_mirror job is not modified.
prod_mirror_dev_verify:
  extends: prod_mirror
  environment:
    name: dev
  variables:
    EL_LABEL: "Mirror DEV Verify-Only"
    EL_ENV_FILE: '\\thqpcelerra01\ttb_common\_OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'
    DBT_TARGET: "dev"
    EL_VERIFY_ONLY: "true"
    DBT_SELECT: "myttb_controlled_group"
  resource_group: postgres-el-dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "pipeline-testing" && $CI_PIPELINE_SOURCE == "web"'
      when: manual
    - when: never
  before_script:
    - uv sync --quiet
    - |
      if (-not (Test-Path -LiteralPath $env:EL_ENV_FILE -PathType Leaf)) { throw "EL_ENV_FILE not accessible" }
      $found = @{}
      foreach ($line in [System.IO.File]::ReadAllLines($env:EL_ENV_FILE)) {
        if ($line -match '^\s*(MIRROR_PG_[A-Z0-9_]+)\s*=\s*(.*?)\s*$') { $found[$matches[1]] = -not [string]::IsNullOrWhiteSpace($matches[2].Trim('"', "'")) }
      }
      if ($found.Count -eq 0) { throw "No MIRROR_PG_* settings in EL_ENV_FILE" }
      foreach ($k in ($found.Keys | Sort-Object)) { Write-Host ("{0} = {1}" -f $k, $(if ($found[$k]) { 'SET' } else { 'EMPTY' })) }

'@ -split "`r?`n"
$L.InsertRange($e, [string[]]$job)
[IO.File]::WriteAllText($p, ($L -join $nl), (New-Object Text.UTF8Encoding($bom)))
$ErrorActionPreference = 'Continue'
$add = @(git diff -U0 -- .gitlab-ci.yml | Where-Object { $_ -match '^\+[^+]' }).Count
$del = @(git diff -U0 -- .gitlab-ci.yml | Where-Object { $_ -match '^-[^-]' }).Count
$exp = @($job | Where-Object { $_ -ne '' }).Count
"Changed lines: +$add -$del (expect +$exp -0)"
if ($add -eq $exp -and $del -eq 0) {
  git add -- .gitlab-ci.yml
  git commit -q -m "Add read-only mirror readiness job (prod_mirror verify-only into DEV, manual)" -m "prod_mirror_dev_verify extends prod_mirror unchanged; checks MIRROR_PG_* key presence (no values) and runs --verify-only for myttb_controlled_group. No writes."
  git push -o ci.skip origin pipeline-testing
  git log --oneline -2
  'RESULT: ADDED, COMMITTED AND PUSHED (no pipeline started)'
} else {
  git checkout -- .gitlab-ci.yml
  'RESULT: unexpected change - reverted, nothing committed. Send me this output.'
}
