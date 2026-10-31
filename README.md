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
if ($L.Contains('postgres_ivv_target_inspect:')) { throw 'STOP: IVV inspect job already exists' }
$t = $L.IndexOf('postgres_extract_load_test:'); if ($t -lt 0) { throw 'STOP: postgres_extract_load_test not found' }
$job = @'
# READ-ONLY IVV target inspection (forced read-only session). Shows whether the
# raw_myttb schema / test tables / load-state exist in IVV and whether the login
# may create tables there. Never prints credentials.
postgres_ivv_target_inspect:
  stage: extract_load
  environment:
    name: ivv
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
      conn = psycopg.connect(host="oa-tst-flexible-server.postgres.database.usgovcloudapi.net", port=5432, dbname="oablmsdb", user=os.environ["DBT_PG_USER"], password=os.environ["DBT_PG_PASSWORD"], options="-c default_transaction_read_only=on", connect_timeout=30)
      cur = conn.cursor()
      cur.execute("SHOW default_transaction_read_only"); print("read_only_session:", cur.fetchone()[0])
      cur.execute("SELECT current_database()"); print("database:", cur.fetchone()[0])
      cur.execute("SELECT has_database_privilege(current_user, current_database(), %s)", ("CREATE",)); print("can_create_schema:", cur.fetchone()[0])
      cur.execute("SELECT to_regnamespace(%s)::text", ("raw_myttb",)); sch = cur.fetchone()[0]; print("schema raw_myttb:", sch)
      if sch:
          cur.execute("SELECT has_schema_privilege(current_user, %s, %s)", ("raw_myttb", "CREATE")); print("can_create_in_raw_myttb:", cur.fetchone()[0])
          cur.execute("SELECT count(*) FROM information_schema.tables WHERE table_schema = %s", ("raw_myttb",)); print("tables_in_raw_myttb:", cur.fetchone()[0])
      for t in ("raw_myttb.myttb_cg_ref_identifier_type", "raw_myttb.myttb_controlled_group"):
          cur.execute("SELECT to_regclass(%s)::text", (t,)); print(t, "->", cur.fetchone()[0])
      cur.execute("SELECT to_regclass(%s)::text", ("extract_load_meta._el_state",)); st = cur.fetchone()[0]; print("state_table:", st)
      if st:
          cur.execute("SELECT target_schema, target_table, cursor_column, high_water_mark, last_run_started_at, status FROM extract_load_meta._el_state ORDER BY last_run_started_at DESC NULLS LAST LIMIT 20")
          for r in cur.fetchall(): print([str(v) for v in r])
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
  git commit -q -m "Add read-only IVV target inspection job (manual, pipeline-testing)" -m "Read-only session on the IVV target; reports raw_myttb schema/table existence, create privileges and load-state rows. No writes."
  git push -o ci.skip origin pipeline-testing
  git log --oneline -2
  'RESULT: ADDED, COMMITTED AND PUSHED (no pipeline started)'
} else {
  git checkout -- .gitlab-ci.yml
  'RESULT: unexpected change - reverted, nothing committed. Send me this output.'
}
