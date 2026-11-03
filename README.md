& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune 2>$null
$yml = Join-Path $env:TEMP 'oa_prod_ci.yml'
git show origin/feature/prod-el-readiness:.gitlab-ci.yml | Set-Content -Path $yml -Encoding UTF8
$py = Join-Path $env:TEMP 'oa_tpl_dump.py'
@'
import sys, yaml
class L(yaml.SafeLoader): pass
L.add_constructor('!reference', lambda l, n: ['!reference'] + l.construct_sequence(n))
ci = list(yaml.load_all(open(sys.argv[1], encoding='utf-8'), Loader=L))[-1]
for name in ['.dbt_env', '.el_upstream', '.el_job', '.dbt_failure_policy', '.skip_if_extract_load_only',
             '.manual_on_development', '.on_any_branch_push', '.on_code_review', '.dbt_lint']:
    t = ci.get(name)
    print(f'\n===== {name} =====')
    if not isinstance(t, dict): print('  (not found)'); continue
    for k in ('extends', 'environment', 'variables', 'allow_failure', 'rules'):
        if k in t:
            v = t[k]
            if k == 'rules':
                print('  rules:')
                for r in v: print('    ', r)
            else:
                print(f'  {k}: {v}')
'@ | Set-Content -Path $py -Encoding UTF8
uv run --no-sync python $py $yml
}
