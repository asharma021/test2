Set-Location C:\Users\SharmaAashishR\oa-business-layer
$ErrorActionPreference = 'Continue'

"--- git status ---";            git status --short
"--- git diff --stat ---";       git diff --stat
"--- git diff -- .gitlab-ci.yml ---"
git diff -- .gitlab-ci.yml

$L = [IO.File]::ReadAllLines((Join-Path $PWD '.gitlab-ci.yml'))
$s = -1; for ($i = 0; $i -lt $L.Count; $i++) { if ($L[$i] -match '^postgres_extract_load:\s*$') { $s = $i } }
$e = $s; for ($i = $s + 1; $i -lt $L.Count; $i++) { if ($L[$i] -match '^\S') { break }; if ($L[$i].Trim() -ne '') { $e = $i } }
$outside = git diff -U0 -- .gitlab-ci.yml | Select-String -Pattern '^@@ -\d+(?:,\d+)? \+(\d+)(?:,(\d+))? @@' | ForEach-Object {
  $start = [int]$_.Matches[0].Groups[1].Value
  $len   = if ($_.Matches[0].Groups[2].Success) { [int]$_.Matches[0].Groups[2].Value } else { 1 }
  if ($len -gt 0 -and ($start -lt ($s + 1) -or ($start + $len - 1) -gt ($e + 1))) { $_.Line }
}
"Block now at lines $($s+1)-$($e+1). Hunks outside block: $(if ($outside) { $outside -join '; ' } else { 'none' })"

@'
import subprocess, sys, yaml

class L(yaml.SafeLoader):
    pass
L.add_constructor('!reference', lambda l, n: {'!reference': l.construct_sequence(n)})

def load(text):
    docs = list(yaml.load_all(text, Loader=L))
    assert len(docs) == 2, 'spec/inputs --- separator missing'
    return docs

def git_show(ref):
    out = subprocess.run(['git', 'show', f'{ref}:.gitlab-ci.yml'], capture_output=True, check=True).stdout
    return out.decode('utf-8-sig')

head = load(git_show('HEAD'))
dev  = load(git_show('origin/development'))
new  = load(open('.gitlab-ci.yml', encoding='utf-8-sig').read())

changed = sorted(k for k in set(head[1]) | set(new[1]) if head[1].get(k) != new[1].get(k))
same_as_dev = new[1].get('postgres_extract_load') == dev[1].get('postgres_extract_load')
print('YAML parses (2 documents)       : True')
print('spec/inputs unchanged           :', head[0] == new[0])
print('Top-level keys changed vs HEAD  :', changed)
print('postgres_extract_load == dev    :', same_as_dev)

protected = ['postgres_extract_load_test', 'postgres_extract_load_dev_incremental',
             'postgres_extract_load_ivv_incremental', 'check_postgres_test_access',
             '.postgres_incremental_guard', '.el_job', '.el_upstream', 'prod_mirror']
all_kept = True
for j in protected:
    ok = j in new[1] and head[1].get(j) == new[1].get(j)
    all_kept &= ok
    print(f'  {j:40} unchanged: {ok}')

print('postgres_extract_load rules (new):')
auto_on_test_branch = False
for r in new[1]['postgres_extract_load']['rules']:
    cond = r.get('if', '<none>'); when = r.get('when', 'on_success')
    print(f'  when={when:10} if={cond}')
    if ('pipeline-testing' in cond or '"development"' in cond) and when not in ('manual', 'never'):
        auto_on_test_branch = True
print('Any automatic rule on pipeline-testing/development:', auto_on_test_branch)

ok = changed == ['postgres_extract_load'] and same_as_dev and head[0] == new[0] and all_kept and not auto_on_test_branch
print('RESTORE CHECK:', 'PASS' if ok else 'FAIL')
sys.exit(0 if ok else 1)
'@ | Set-Content -Encoding utf8 (Join-Path $env:TEMP 'check_restore.py')
uv run --no-sync python (Join-Path $env:TEMP 'check_restore.py')
