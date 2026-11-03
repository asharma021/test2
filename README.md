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
$c = [IO.File]::ReadAllText($cp)
if ($c.Contains('CI_MERGE_REQUEST_SOURCE_BRANCH_PROTECTED')) { Undo "rule already present"; return }
$nl = if ($c.Contains("`r`n")) { "`r`n" } else { "`n" }
$lines = New-Object System.Collections.ArrayList
$null = $lines.AddRange(($c -split "\r?\n"))
$w = @(0..($lines.Count-1) | Where-Object { $lines[$_] -eq 'workflow:' })
if ($w.Count -ne 1) { Undo "'workflow:' found $($w.Count) times"; return }
$e = $w[0] + 1
while ($e -lt $lines.Count -and ($lines[$e] -match '^\s' -or $lines[$e] -eq '')) { $e++ }
$r = @(($w[0]+1)..($e-1) | Where-Object { $lines[$_] -match '^\s+rules:\s*$' })
if ($r.Count -ne 1) { Undo "workflow 'rules:' not found exactly once"; return }
$first = @(($r[0]+1)..($e-1) | Where-Object { $lines[$_] -match '^\s+- ' } | Select-Object -First 1)
if (-not $first) { Undo "no existing workflow rule found"; return }
$ind = $lines[$first[0]] -replace '^(\s*)-.*$', '$1'
$new = @(
  $ind + '# Unprotected source branch: no MR pipeline (the protected runner cannot run it)',
  $ind + '- if: ''$CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_SOURCE_BRANCH_PROTECTED != "true"''',
  $ind + '  when: never',
  $ind + '# Unprotected branch push/run: no pipeline; it runs after merge into a protected branch',
  $ind + '- if: ''$CI_PIPELINE_SOURCE != "merge_request_event" && $CI_COMMIT_REF_PROTECTED != "true"''',
  $ind + '  when: never'
)
$lines.InsertRange($first[0], $new)
[IO.File]::WriteAllText($cp, ($lines -join $nl), (New-Object Text.UTF8Encoding($false)))

$origYml = Join-Path $env:TEMP 'oa_ci_orig.yml'
git show origin/development:.gitlab-ci.yml | Set-Content -Path $origYml -Encoding UTF8
$py = Join-Path $env:TEMP 'oa_validate_workflow.py'
@'
import sys, yaml
class L(yaml.SafeLoader): pass
L.add_constructor('!reference', lambda l, n: ['!reference'] + l.construct_sequence(n))
new = list(yaml.load_all(open(sys.argv[1], encoding='utf-8-sig'), Loader=L))[-1]['workflow']['rules']
old = list(yaml.load_all(open(sys.argv[2], encoding='utf-8-sig'), Loader=L))[-1]['workflow']['rules']
assert new[0] == {'if': '$CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_SOURCE_BRANCH_PROTECTED != "true"', 'when': 'never'}, new[0]
assert new[1] == {'if': '$CI_PIPELINE_SOURCE != "merge_request_event" && $CI_COMMIT_REF_PROTECTED != "true"', 'when': 'never'}, new[1]
assert new[2:] == old, ('existing rules changed', new[2:], old)
print('OK new rule 1:', new[0]); print('OK new rule 2:', new[1]); print('OK existing workflow rules unchanged:', len(old), 'rule(s)')
'@ | Set-Content -Path $py -Encoding UTF8
uv run --no-sync python $py $cp $origYml
if ($LASTEXITCODE -ne 0) { Undo "validation failed"; return }
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
