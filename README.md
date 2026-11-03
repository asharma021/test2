& {
$repo = 'C:\Users\SharmaAashishR\oa-business-layer'
Set-Location $repo
$branch = 'ci/skip-unprotected-pipelines'
$orig = (git rev-parse --abbrev-ref HEAD).Trim()
function Undo([string]$msg) {
  Write-Host "FAILED: $msg" -ForegroundColor Red
  git reset --hard --quiet 2>$null
  git switch --quiet $orig 2>$null
  if ((git rev-parse --abbrev-ref HEAD).Trim() -eq $orig) { git branch -D $branch 2>$null | Out-Null; Write-Host "Reverted: back on '$orig', '$branch' removed. Nothing pushed." -ForegroundColor Yellow }
}
if (git status --porcelain --untracked-files=no) { Write-Host "Uncommitted changes - nothing done." -ForegroundColor Red; return }
git fetch origin --prune 2>$null
if (git branch --list $branch) { Write-Host "Local $branch exists - nothing done." -ForegroundColor Red; return }
if (git ls-remote --heads origin $branch) { Write-Host "Remote $branch exists - nothing done." -ForegroundColor Red; return }
git switch --quiet -c $branch origin/development
if ($LASTEXITCODE -ne 0) { Write-Host "Could not create branch - nothing done." -ForegroundColor Red; return }

$cp = Join-Path $repo '.gitlab-ci.yml'
$origYml = Join-Path $env:TEMP 'oa_ci_orig.yml'
Copy-Item $cp $origYml -Force
$py = Join-Path $env:TEMP 'oa_edit_workflow.py'
@'
import sys, yaml
p, orig = sys.argv[1], sys.argv[2]
raw = open(p, 'rb').read().decode('utf-8')
bom = raw.startswith('\ufeff')
if bom: raw = raw[1:]
if 'CI_MERGE_REQUEST_SOURCE_BRANCH_PROTECTED' in raw: sys.exit('rule already present')
nl = '\r\n' if '\r\n' in raw else '\n'
lines = raw.split(nl)
idx = [i for i, l in enumerate(lines) if l == 'workflow:']
if len(idx) != 1: sys.exit(f"'workflow:' found {len(idx)} times")
w = idx[0]
if lines[w+1] != '  rules:': sys.exit(f'unexpected line after workflow: {lines[w+1]!r}')
if not lines[w+2].startswith('    - '): sys.exit(f'unexpected first rule line: {lines[w+2]!r}')
new = [
    "    # Unprotected source branch: no MR pipeline (the protected runner cannot run it)",
    "    - if: '$CI_PIPELINE_SOURCE == \"merge_request_event\" && $CI_MERGE_REQUEST_SOURCE_BRANCH_PROTECTED != \"true\"'",
    "      when: never",
    "    # Unprotected branch push/run: no pipeline; it runs after merge into a protected branch",
    "    - if: '$CI_PIPELINE_SOURCE != \"merge_request_event\" && $CI_COMMIT_REF_PROTECTED != \"true\"'",
    "      when: never",
]
lines[w+2:w+2] = new
open(p, 'wb').write((('\ufeff' if bom else '') + nl.join(lines)).encode('utf-8'))
class L(yaml.SafeLoader): pass
L.add_constructor('!reference', lambda l, n: ['!reference'] + l.construct_sequence(n))
def wf(f): return list(yaml.load_all(open(f, encoding='utf-8-sig'), Loader=L))[-1]['workflow']['rules']
n, o = wf(p), wf(orig)
assert n[0] == {'if': '$CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_SOURCE_BRANCH_PROTECTED != "true"', 'when': 'never'}, n[0]
assert n[1] == {'if': '$CI_PIPELINE_SOURCE != "merge_request_event" && $CI_COMMIT_REF_PROTECTED != "true"', 'when': 'never'}, n[1]
assert n[2:] == o, ('existing rules changed', n[2:], o)
print('OK new rule 1:', n[0]); print('OK new rule 2:', n[1]); print('OK existing workflow rules unchanged:', len(o), 'rule(s)')
'@ | Set-Content -Path $py -Encoding UTF8
uv run --no-sync python $py $cp $origYml
if ($LASTEXITCODE -ne 0) { Undo "edit/validation failed - see message above"; return }
$changed = @(git diff --name-only)
if (($changed -join '|') -ne '.gitlab-ci.yml') { Undo "unexpected files: $($changed -join ', ')"; return }
Write-Host "`n===== DIFF =====" -ForegroundColor Cyan
git --no-pager diff

$msgFile = Join-Path $env:TEMP 'oa_workflow_msg.txt'
$msg = "Skip pipelines for unprotected branches`n`nThe only runner is protected, so pipelines on unprotected branches and their`nMR pipelines were created but never picked up (stuck). They are now not`ncreated at all. Protected branches (development, remote-development, test,`nproduction, pipeline-testing) and MRs between protected branches are`nunchanged, and a pipeline still runs after a merge into a protected branch.`nPre-merge checks come from the local pre-commit hooks.`n"
[IO.File]::WriteAllText($msgFile, $msg, (New-Object Text.UTF8Encoding($false)))
git add .gitlab-ci.yml
git commit --quiet -F $msgFile
if ($LASTEXITCODE -ne 0) { Undo "commit failed (hook?) - see output above"; return }
git push -u origin $branch -o ci.skip -o merge_request.create -o merge_request.target=development -o "merge_request.title=Skip pipelines for unprotected branches (no more stuck pipelines)"
if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - commit kept locally on $branch; send me the error." -ForegroundColor Red; return }
Write-Host "DONE: $(git log -1 --format='%h %s') pushed; MR into development created." -ForegroundColor Green
git switch --quiet $orig
Write-Host "Back on $(git rev-parse --abbrev-ref HEAD)." -ForegroundColor Green
}
