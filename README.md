& {
$repo = 'C:\Users\SharmaAashishR\oa-business-layer'
Set-Location $repo
$branch = 'feature/prod-el-readiness'
$orig = (git rev-parse --abbrev-ref HEAD).Trim()
function Undo([string]$msg) {
  Write-Host "FAILED: $msg" -ForegroundColor Red
  git checkout --quiet -- . 2>$null
  git switch --quiet $orig 2>$null
  git branch -D $branch 2>$null | Out-Null
  Write-Host "Reverted: back on '$orig', '$branch' removed. Nothing pushed." -ForegroundColor Yellow
}
if (git status --porcelain --untracked-files=no) { Write-Host "Uncommitted changes on '$orig' - nothing done." -ForegroundColor Red; return }
git fetch origin --prune 2>$null
if (git branch --list $branch) { Write-Host "Local branch $branch already exists - nothing done." -ForegroundColor Red; return }
if (git ls-remote --heads origin $branch) { Write-Host "Remote branch $branch already exists - nothing done." -ForegroundColor Red; return }
git switch --quiet -c $branch origin/development
if ($LASTEXITCODE -ne 0) { Undo "could not create branch"; return }
$utf8 = New-Object Text.UTF8Encoding($false)

# --- 1) dbt/profiles.yml prod target ---
$pp = Join-Path $repo 'dbt\profiles.yml'
$p = [IO.File]::ReadAllText($pp)
$rxHost = '(?m)^([ \t]+)host:[ \t]*TODO-prod-host\.postgres\.database\.usgovcloudapi\.net[ \t]*(?=\r?$)'
$rxDb   = '(?m)^([ \t]+)dbname:[ \t]*business_layer_prod[ \t]*(?=\r?$)'
if ([regex]::Matches($p, $rxHost).Count -ne 1) { Undo "prod TODO host line not found exactly once"; return }
if ([regex]::Matches($p, $rxDb).Count -ne 1)   { Undo "prod dbname line not found exactly once"; return }
$p = [regex]::Replace($p, $rxHost, '${1}host: oa-prod-flexible-server.postgres.database.usgovcloudapi.net')
$p = [regex]::Replace($p, $rxDb,   '${1}dbname: oablmsdb')
[IO.File]::WriteAllText($pp, $p, $utf8)

# --- 2) .gitlab-ci.yml: postgres_extract_load default-branch rule -> manual + single table ---
$cp = Join-Path $repo '.gitlab-ci.yml'
$c = [IO.File]::ReadAllText($cp)
$nl = if ($c -match "`r`n") { "`r`n" } else { "`n" }
$lines = New-Object System.Collections.ArrayList
$null = $lines.AddRange(($c -split "\r?\n"))
$starts = @(0..($lines.Count-1) | Where-Object { $lines[$_] -eq 'postgres_extract_load:' })
if ($starts.Count -ne 1) { Undo "postgres_extract_load: found $($starts.Count) times"; return }
$s = $starts[0]; $e = $s + 1
while ($e -lt $lines.Count -and ($lines[$e] -match '^\s' -or $lines[$e] -eq '')) { $e++ }
$hits = @(($s+1)..($e-1) | Where-Object { $lines[$_] -match '\$CI_COMMIT_BRANCH == \$CI_DEFAULT_BRANCH' })
if ($hits.Count -ne 1) { Undo "default-branch rule found $($hits.Count) times in postgres_extract_load"; return }
$r = $hits[0]
if ($lines[$r+1] -notmatch '^\s+variables:\s*$') { Undo "unexpected line after default-branch rule: '$($lines[$r+1])'"; return }
if ($lines[$r+2] -notmatch '^\s+DBT_TARGET:\s*"prod"\s*$') { Undo "expected DBT_TARGET prod, got '$($lines[$r+2])'"; return }
if ($lines[$r+3] -match 'when:|DBT_SELECT') { Undo "rule already modified - stopping"; return }
$vi = $lines[$r+2] -replace '^(\s*).*$', '$1'
$wi = $lines[$r+1] -replace '^(\s*).*$', '$1'
$lines.Insert($r+3, $wi + 'when: manual')
$lines.Insert($r+3, $vi + 'DBT_SELECT: "myttb_cg_ref_identifier_type"')
[IO.File]::WriteAllText($cp, ($lines -join $nl), $utf8)

# --- 3) validate YAML + intended values ---
$py = Join-Path $env:TEMP 'oa_validate_prod.py'
@'
import sys, yaml
class L(yaml.SafeLoader): pass
L.add_constructor('!reference', lambda l, n: l.construct_sequence(n))
ci, prof = sys.argv[1], sys.argv[2]
docs = list(yaml.load_all(open(ci, encoding='utf-8'), Loader=L))
job = docs[-1]['postgres_extract_load']
rule = [x for x in job['rules'] if 'CI_DEFAULT_BRANCH' in str(x.get('if', ''))]
assert len(rule) == 1, rule
rule = rule[0]
assert rule.get('when') == 'manual', rule
assert rule['variables'] == {'DBT_TARGET': 'prod', 'DBT_SELECT': 'myttb_cg_ref_identifier_type'}, rule
print('CI YAML OK  | prod rule:', rule)
def find(o):
    if isinstance(o, dict):
        if isinstance(o.get('prod'), dict) and 'dbname' in o['prod']: return o['prod']
        for v in o.values():
            f = find(v)
            if f: return f
d = find(yaml.load(open(prof, encoding='utf-8'), Loader=L))
assert d and d['dbname'] == 'oablmsdb' and d['schema'] == 'business_layer_prod' and d['host'].startswith('oa-prod-flexible-server'), d
print('profiles OK | prod host:', d['host'], '| dbname:', d['dbname'], '| schema:', d['schema'])
'@ | Set-Content -Path $py -Encoding UTF8
uv run --no-sync python $py $cp $pp
if ($LASTEXITCODE -ne 0) { Undo "YAML validation failed"; return }
$changed = @(git diff --name-only)
if ($changed.Count -ne 2) { Undo "expected 2 changed files, got: $($changed -join ', ')"; return }
Write-Host "`n===== DIFF =====" -ForegroundColor Cyan
git --no-pager diff

# --- 4) commit + push (ci.skip) + draft MR into development ---
$msgFile = Join-Path $env:TEMP 'oa_commit_msg.txt'
@'
Prepare prod PostgreSQL E/L: fix prod target, make prod run manual

- dbt/profiles.yml (prod): dbname business_layer_prod -> oablmsdb
  (business_layer_prod is the schema, per Josh); TODO host placeholder ->
  oa-prod-flexible-server (same server the mirror reads from; PS to confirm).
- .gitlab-ci.yml postgres_extract_load: default-branch (production) rule is
  now when: manual with DBT_SELECT defaulting to myttb_cg_ref_identifier_type,
  so the first prod load is a deliberate single-table run instead of an
  automatic unscoped load on every production pipeline.

No production branch, schedule, credential, or data changes.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01Q5V13xaQ2dZQoLLE4XZbfJ
'@ | Set-Content -Path $msgFile -Encoding UTF8
git add dbt/profiles.yml .gitlab-ci.yml
git commit --quiet -F $msgFile
if ($LASTEXITCODE -ne 0) { Undo "commit failed"; return }
git push -u origin $branch -o ci.skip -o merge_request.create -o merge_request.target=development -o merge_request.draft -o "merge_request.title=Draft: Prep prod PostgreSQL E/L - prod target fix + manual single-table prod run"
if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - commit kept locally on $branch; send me the error." -ForegroundColor Red; return }
Write-Host "`nDONE: $(git log -1 --format='%h %s') pushed; draft MR into development created." -ForegroundColor Green
git switch --quiet $orig
Write-Host "Back on '$orig'." -ForegroundColor Green
}
