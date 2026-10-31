notepad "$env:TEMP\phaseA.ps1"


# ===== PHASE A - READ-ONLY CONFIGURATION SNAPSHOT (modifies nothing) =====
Set-Location C:\Users\SharmaAashishR\oa-business-layer
$ErrorActionPreference = 'Continue'
git fetch origin --prune 2>&1 | Out-Null

function Get-JobBlock([string[]]$L, [string]$name) {
  $s = -1
  for ($i = 0; $i -lt $L.Count; $i++) { if ($L[$i] -match ('^' + [regex]::Escape($name) + ':\s*$')) { $s = $i; break } }
  if ($s -lt 0) { return $null }
  $e = $s
  for ($i = $s + 1; $i -lt $L.Count; $i++) { if ($L[$i] -match '^\S') { break }; if ($L[$i].Trim() -ne '') { $e = $i } }
  return ,([string[]]$L[$s..$e])
}
function Get-Ref([string]$ref) { @(git show "${ref}:.gitlab-ci.yml" | ForEach-Object { $_.TrimEnd("`r") }) }

$pt  = Get-Ref 'origin/pipeline-testing'
$dev = Get-Ref 'origin/development'

$report = & {
  '===== 1. Branch and working tree ====='
  "Branch: $(git branch --show-current)"
  git status --short
  "Uncommitted .gitlab-ci.yml changes: $(if (git status --porcelain -- .gitlab-ci.yml) { 'YES' } else { 'none' })"

  '===== 2. Local HEAD vs origin/pipeline-testing ====='
  "HEAD                   : $(git rev-parse --short HEAD)"
  "origin/pipeline-testing: $(git rev-parse --short origin/pipeline-testing)"
  "origin/development     : $(git rev-parse --short origin/development)"
  "Ahead/behind (HEAD...origin/pipeline-testing): $(git rev-list --left-right --count HEAD...origin/pipeline-testing)"
  git log --oneline -5 origin/pipeline-testing

  '===== 3. PostgreSQL EL related jobs/templates on origin/pipeline-testing ====='
  $pt | Select-String -Pattern '^(\.?postgres_\w+|check_postgres_\w+|prod_mirror|\.el_job|\.el_upstream|\.dbt_env):' |
    ForEach-Object { "{0,5}: {1}" -f $_.LineNumber, $_.Line }

  '===== 4. Job definitions: origin/pipeline-testing vs origin/development ====='
  $jobs = '.dbt_env', '.el_job', '.el_upstream', 'postgres_extract_load', 'postgres_extract_load_test',
          'check_postgres_test_access', '.postgres_incremental_guard',
          'postgres_extract_load_dev_incremental', 'postgres_extract_load_ivv_incremental', 'prod_mirror'
  foreach ($j in $jobs) {
    $a = Get-JobBlock $pt $j; $b = Get-JobBlock $dev $j
    $state = if (-not $a) { 'MISSING on pipeline-testing' }
             elseif (-not $b) { 'only on pipeline-testing (not yet in development)' }
             elseif (($a -join "`n") -ceq ($b -join "`n")) { 'identical to development' }
             else { 'DIFFERS from development' }
    "{0,-40} {1}" -f $j, $state
    if ($a -and $b -and (($a -join "`n") -cne ($b -join "`n"))) {
      Compare-Object $b $a | ForEach-Object { "      {0} {1}" -f ($(if ($_.SideIndicator -eq '<=') { 'development:' } else { 'pipe-test  :' })), $_.InputObject }
    }
  }

  '===== 5. Full PostgreSQL job blocks on origin/pipeline-testing (variables, environment, rules) ====='
  foreach ($j in 'postgres_extract_load', 'postgres_extract_load_test', 'postgres_extract_load_dev_incremental',
                 'postgres_extract_load_ivv_incremental', 'check_postgres_test_access', 'prod_mirror') {
    "----- $j -----"
    $blk = Get-JobBlock $pt $j
    if ($blk) { $blk } else { '(not found)' }
  }

  '===== 6. dbt/profiles.yml targets (non-secret fields only) ====='
  $prof = Join-Path $PWD 'dbt\profiles.yml'
  if (Test-Path $prof) {
    Select-String -Path $prof -Pattern '^\s*(dev|test|prod|evaluator|outputs|target):|^\s*(type|host|port|dbname|database|schema):' |
      Where-Object { $_.Line -notmatch '(?i)user|pass|secret|token' } |
      ForEach-Object { "{0,4}: {1}" -f $_.LineNumber, $_.Line.TrimEnd() }
  } else { 'dbt\profiles.yml not found' }

  '===== 7. EL env files referenced in YAML: accessibility + key NAMES only (no values) ====='
  $paths = $pt | Select-String -Pattern "EL_ENV_FILE:\s*'([^']+)'" | ForEach-Object { $_.Matches[0].Groups[1].Value } | Sort-Object -Unique
  foreach ($p in $paths) {
    $ok = Test-Path -LiteralPath $p -PathType Leaf
    "$p  ->  accessible from this machine: $ok"
    if ($ok) {
      foreach ($line in [IO.File]::ReadAllLines($p)) {
        if ($line -match '^\s*([A-Z][A-Z0-9_]*)\s*=\s*(.*)$') {
          "      {0,-32} {1}" -f $matches[1], $(if ([string]::IsNullOrWhiteSpace($matches[2].Trim('"', "'"))) { 'EMPTY' } else { 'SET' })
        }
      }
    }
  }

  '===== 8. Restore status ====='
  "Restore commit on origin/pipeline-testing: $(git log --oneline --grep 'Restore approved postgres_extract_load' origin/pipeline-testing)"
  $a = Get-JobBlock $pt 'postgres_extract_load'; $b = Get-JobBlock $dev 'postgres_extract_load'
  "postgres_extract_load committed on pipeline-testing identical to development: $(($a -join "`n") -ceq ($b -join "`n"))"

  '===== 9. Protected branches ====='
  'Not visible from git. Check in GitLab: Settings > Repository > Protected branches (pipeline-testing, development, production).'
}

$out = Join-Path $env:TEMP 'phaseA_report.txt'
$report | Out-File -Encoding utf8 $out
"Report written to $out"
notepad $out

















