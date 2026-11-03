& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
$yml = Join-Path $env:TEMP 'oa_prod_ci.yml'
git show origin/feature/prod-el-readiness:.gitlab-ci.yml | Set-Content -Path $yml -Encoding UTF8
$py = Join-Path $env:TEMP 'oa_rules_dump.py'
@'
import sys, yaml
class L(yaml.SafeLoader): pass
L.add_constructor('!reference', lambda l, n: ['!reference'] + l.construct_sequence(n))
docs = list(yaml.load_all(open(sys.argv[1], encoding='utf-8'), Loader=L))
ci = docs[-1]
print('=== workflow ===')
print(ci.get('workflow'))
print('=== global variables (DBT_*/EL_*) ===')
for k, v in (ci.get('variables') or {}).items():
    if k.startswith(('DBT_', 'EL_')): print(f'  {k} = {v}')
print('=== jobs: rules ===')
for name, job in ci.items():
    if name.startswith('.') or not isinstance(job, dict) or name in ('workflow', 'variables', 'default', 'stages', 'include', 'spec'): continue
    print(f'\n[{name}] stage={job.get("stage")} extends={job.get("extends")} env={job.get("environment")} allow_failure={job.get("allow_failure")}')
    for r in job.get('rules') or []:
        print('   ', r)
'@ | Set-Content -Path $py -Encoding UTF8
uv run --no-sync python $py $yml
}
