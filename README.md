# ---- 0. Context (fill in ProjectPath) ----
$ProjectPath = '<group>/oa-business-layer'
$GitLab      = 'https://gitlab.ad.ttb.gov'
$ProjEnc     = [uri]::EscapeDataString($ProjectPath)
$Hdr         = @{ 'PRIVATE-TOKEN' = $env:GITLAB_TOKEN }
$Ci          = Join-Path (Get-Location) '.gitlab-ci.yml'

# ---- 1. Only postgres_extract_load changed ----
git diff -U0 -- .gitlab-ci.yml

# ---- 2. No double-quoted UNC paths remain ----
Select-String -Path $Ci -SimpleMatch '"\\thqpcelerra01' |
  ForEach-Object { "DOUBLE-QUOTED UNC at line $($_.LineNumber): $($_.Line.Trim())" }
'UNC quote check complete'

# ---- 3. postgres_extract_load structural check ----
@'
import sys, yaml
class L(yaml.SafeLoader): pass
L.add_constructor('!reference', lambda l, n: {'!reference': l.construct_sequence(n)})
docs = list(yaml.load_all(open(sys.argv[1], encoding='utf-8-sig'), Loader=L))
assert len(docs) == 2, 'spec/inputs --- separator missing'
ci = docs[1]
def flat(rules):
    out = []
    for r in rules:
        if isinstance(r, dict) and '!reference' in r:
            ref = r['!reference']; out += flat(ci[ref[0]][ref[1]])
        else:
            out.append(r)
    return out
job = ci['postgres_extract_load']
risky = {'EL_VERIFY_ONLY','DBT_SELECT','EL_CONFIG','EL_ENV_FILE','DBT_TARGET','DBT_LOAD_MODE'}
leak = sorted(risky & set(job.get('variables', {})))
print('Job-level variables    :', job.get('variables'))
print('Test vars at job level :', leak or 'none')
rules = flat(job['rules'])
for i, r in enumerate(rules, 1):
    print(f'Rule {i}: if={r.get("if")!r} when={r.get("when","on_success")} vars={r.get("variables")}')
prod = next(r for r in rules if r.get('if','').strip("'").endswith('$CI_DEFAULT_BRANCH'))
print('Prod rule variables    :', prod.get('variables'))
test = next(r for r in rules if 'pipeline-testing' in r.get('if',''))
print('EL_ENV_FILE parsed as  :', test['variables']['EL_ENV_FILE'])
ok = not leak and prod.get('variables') == {'DBT_TARGET': 'prod'}
print('PHASE 1 CHECK:', 'PASS' if ok else 'FAIL')
'@ | Set-Content -Encoding utf8 "$env:TEMP\check_p1.py"
uv run python "$env:TEMP\check_p1.py" $Ci

# ---- 4. Approved prod postgres_extract_load on the default branch ----
$DefaultBranch = (Invoke-RestMethod -Uri "$GitLab/api/v4/projects/$ProjEnc" -Headers $Hdr).default_branch
"Default branch: $DefaultBranch"
$prodCi = git show "origin/${DefaultBranch}:.gitlab-ci.yml"
$s = ($prodCi | Select-String -Pattern '^postgres_extract_load:').LineNumber
if ($s) { $prodCi[($s-1)..($s+8)] } else { 'postgres_extract_load not found on default branch' }

# ---- 5. CI Lint dry run - PostgreSQL jobs only ----
$content = Get-Content $Ci -Raw
foreach ($ref in @('pipeline-testing', $DefaultBranch)) {
  $body = @{ content = $content; dry_run = $true; include_jobs = $true; ref = $ref } | ConvertTo-Json -Depth 5
  $lint = Invoke-RestMethod -Method Post -Uri "$GitLab/api/v4/projects/$ProjEnc/ci/lint" -Headers $Hdr -ContentType 'application/json' -Body $body
  "`n=== Lint ref=$ref  valid=$($lint.valid) ==="
  $lint.errors; $lint.warnings
  $lint.jobs | Where-Object { $_.name -like '*postgres*' } |
    Select-Object name, stage, when, allow_failure | Format-Table -AutoSize
}
