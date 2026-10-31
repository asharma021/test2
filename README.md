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
if ($L.Contains('mirror_coverage_inspect:')) { throw 'STOP: mirror_coverage_inspect already exists' }
$t = $L.IndexOf('postgres_extract_load_test:'); if ($t -lt 0) { throw 'STOP: postgres_extract_load_test not found' }
$job = @'
# READ-ONLY mirror coverage inspection: catalog metadata only (table names and
# planner row estimates, no table data) from prod OA (mirror read-only role) and
# DEV OA, both in forced read-only sessions. Never prints credentials.
mirror_coverage_inspect:
  stage: extract_load
  environment:
    name: dev
  tags:
    - python-app05
  variables:
    EL_ENV_FILE: '\\thqpcelerra01\ttb_common\_OA_Data_Sharing\OA\dbt_dev_env\extract_load.env'
  rules:
    - if: '$CI_COMMIT_BRANCH == "pipeline-testing" && $CI_PIPELINE_SOURCE == "web"'
      when: manual
    - when: never
  script:
    - uv sync --quiet
    - |
      $code = '
      import os, psycopg
      e = {}
      for line in open(os.environ["EL_ENV_FILE"], encoding="utf-8-sig"):
          line = line.strip()
          if line and not line.startswith("#") and "=" in line:
              k, v = line.split("=", 1); e[k.strip()] = v.strip().strip("\"").strip(chr(39))
      ro = "-c default_transaction_read_only=on"
      prod = psycopg.connect(host=e["MIRROR_PG_HOST"], port=int(e.get("MIRROR_PG_PORT") or 5432), dbname=e["MIRROR_PG_DATABASE"], user=e["MIRROR_PG_USER"], password=e["MIRROR_PG_PASSWORD"], options=ro, connect_timeout=30)
      dev = psycopg.connect(host="oa-dev-flexible-server.postgres.database.usgovcloudapi.net", port=5432, dbname="oablmsdb", user=os.environ["DBT_PG_USER"], password=os.environ["DBT_PG_PASSWORD"], options=ro, connect_timeout=30)
      def tables(conn):
          cur = conn.cursor()
          cur.execute("SELECT n.nspname, c.relname, c.reltuples::bigint FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace WHERE n.nspname LIKE %s AND c.relkind IN (%s, %s)", ("raw\\_%", "r", "p"))
          return {f"{s}.{t}": r for s, t, r in cur.fetchall()}
      P = tables(prod); D = tables(dev)
      both = sorted(set(P) & set(D))
      print("prod_raw_tables:", len(P), "dev_raw_tables:", len(D), "in_both:", len(both))
      print("prod_raw_schemas:", sorted({t.split(".")[0] for t in P}))
      print("myttb_in_prod:", sorted(t for t in P if "myttb" in t)[:20])
      print("only_in_dev_first15:", sorted(set(D) - set(P))[:15])
      for t in sorted(both, key=lambda x: max(P[x], 0))[:15]:
          print("candidate", t, "prod_est", P[t], "dev_est", D[t])
      prod.rollback(); dev.rollback(); prod.close(); dev.close()
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
  git commit -q -m "Add read-only mirror coverage inspection job (manual, pipeline-testing)" -m "Catalog metadata only (table names, planner row estimates) from prod OA via the mirror read-only role and from DEV OA; forced read-only sessions; no table data, no writes."
  git push -o ci.skip origin pipeline-testing
  git log --oneline -2
  'RESULT: ADDED, COMMITTED AND PUSHED (no pipeline started)'
} else {
  git checkout -- .gitlab-ci.yml
  'RESULT: unexpected change - reverted, nothing committed. Send me this output.'
}
