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
if ($L.Contains('postgres_dev_target_inspect:')) { throw 'STOP: inspect job already exists' }
$t = $L.IndexOf('postgres_extract_load_test:'); if ($t -lt 0) { throw 'STOP: postgres_extract_load_test not found' }
$job = @'
# READ-ONLY DEV target inspection for the incremental test table. The session is
# forced read-only (default_transaction_read_only=on); prints counts, duplicate ids
# and the stored watermark row. Never prints credentials.
postgres_dev_target_inspect:
  stage: extract_load
  environment:
    name: dev
  tags:
    - python-app05
  rules:
    - if: '$CI_COMMIT_BRANCH == "pipeline-testing" && $CI_PIPELINE_SOURCE == "web"'
      when: manual
    - when: never
  script:
    - uv sync --quiet
    - |
      $code = '
      import os, psycopg
      conn = psycopg.connect(host="oa-dev-flexible-server.postgres.database.usgovcloudapi.net", port=5432, dbname="oablmsdb", user=os.environ["DBT_PG_USER"], password=os.environ["DBT_PG_PASSWORD"], options="-c default_transaction_read_only=on", connect_timeout=30)
      cur = conn.cursor()
      cur.execute("SHOW default_transaction_read_only"); print("read_only_session:", cur.fetchone()[0])
      cur.execute("SELECT to_regclass(%s)::text", ("raw_myttb.myttb_controlled_group",)); print("target_table:", cur.fetchone()[0])
      cur.execute("SELECT count(*), count(DISTINCT id), count(*) - count(DISTINCT id), min(last_modified_date), max(last_modified_date) FROM raw_myttb.myttb_controlled_group")
      print("rows, distinct_ids, duplicate_rows, min_cursor, max_cursor:", cur.fetchone())
      cur.execute("SELECT to_regclass(%s)::text", ("extract_load_meta._el_state",)); st = cur.fetchone()[0]; print("state_table:", st)
      if st:
          cur.execute("SELECT * FROM extract_load_meta._el_state s WHERE s::text ILIKE %s", ("%myttb_controlled_group%",))
          cols = [d.name for d in cur.description]
          rows = cur.fetchall(); print("state_rows:", len(rows))
          for r in rows: print({c: str(v) for c, v in zip(cols, r)})
      conn.rollback(); conn.close()
      '
      $code | uv run python -
      if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

'@ -split "`r?`n"
$L.InsertRange($t, [string[]]$job)
[IO.File]::WriteAllText($p, ($L -join $nl), (New-Object Text.UTF8Encoding($bom)))
$ErrorActionPreference = 'Continue'
$add = @(git diff -U0 -- .gitlab-ci.yml | Where-Object { $_ -match '^\+[^+]' }).Count
$del = @(git diff -U0 -- .gitlab-ci.yml | Where-Object { $_ -match '^-[^-]' }).Count
$exp = @($job | Where-Object { $_ -ne '' }).Count
"Changed lines: +$add -$del (expect +$exp -0)"
if ($add -eq $exp -and $del -eq 0) {
  git add -- .gitlab-ci.yml
  git commit -q -m "Add read-only DEV target inspection job (manual, pipeline-testing)" -m "Read-only session; reports row count, distinct ids, duplicate rows, cursor range and stored watermark for raw_myttb.myttb_controlled_group. No writes."
  git push -o ci.skip origin pipeline-testing
  git log --oneline -2
  'RESULT: ADDED, COMMITTED AND PUSHED (no pipeline started)'
} else {
  git checkout -- .gitlab-ci.yml
  'RESULT: unexpected change - reverted, nothing committed. Send me this output.'
}
