Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune

# --- Current branch / working tree ---
"Branch: $(git branch --show-current)"
git status --short
"HEAD == origin/pipeline-testing: $((git rev-parse HEAD) -eq (git rev-parse origin/pipeline-testing))"

# --- Merge history ---
git log --oneline -5 origin/development
git log --oneline -5 origin/pipeline-testing
git merge-base --is-ancestor origin/development origin/pipeline-testing
"development contained in pipeline-testing: $($LASTEXITCODE -eq 0)"
"--- Commits on pipeline-testing not yet in development ---"
git log --oneline origin/development..origin/pipeline-testing
"--- Commits on development not yet in pipeline-testing ---"
git log --oneline origin/pipeline-testing..origin/development

# --- PostgreSQL jobs present on each branch ---
foreach ($b in 'development', 'pipeline-testing') {
  "`n== origin/$b =="
  git show "origin/${b}:.gitlab-ci.yml" |
    Select-String -Pattern '^(\.?postgres_\w+|check_postgres_\w+|prod_mirror):' |
    ForEach-Object { "{0,5}: {1}" -f $_.LineNumber, $_.Line }
}

# --- Approved postgres_extract_load block on development ---
$f = git show "origin/development:.gitlab-ci.yml"
$s = ($f | Select-String -Pattern '^postgres_extract_load:').LineNumber
if ($s) { $f[($s-1)..($s+30)] } else { 'postgres_extract_load not found on development' }

# --- How EL_ENV_FILE is actually written (quoting check) ---
git show "origin/pipeline-testing:.gitlab-ci.yml" | Select-String -Pattern 'EL_ENV_FILE:' |
  ForEach-Object { "{0,5}: {1}" -f $_.LineNumber, $_.Line.Trim() }

# --- PostgreSQL CI differences between development and pipeline-testing ---
git diff origin/development origin/pipeline-testing -- .gitlab-ci.yml
