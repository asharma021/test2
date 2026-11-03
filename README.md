& {
$repo = 'C:\Users\SharmaAashishR\oa-business-layer'
Set-Location $repo
$branch = 'fix/abbrev-test-compile'
$orig = (git rev-parse --abbrev-ref HEAD).Trim()
function Undo([string]$msg) {
  Write-Host "FAILED: $msg" -ForegroundColor Red
  git reset --hard --quiet 2>$null
  git switch --quiet $orig 2>$null
  if ((git rev-parse --abbrev-ref HEAD).Trim() -eq $orig) { git branch -D $branch 2>$null | Out-Null; Write-Host "Reverted: back on '$orig', '$branch' removed. Nothing pushed." -ForegroundColor Yellow }
  else { Write-Host "Could not switch back to '$orig' - send me this output." -ForegroundColor Red }
}
if (git status --porcelain --untracked-files=no) { Write-Host "Uncommitted changes on '$orig' - nothing done." -ForegroundColor Red; return }
git fetch origin --prune 2>$null
if (git branch --list $branch) { Write-Host "Local branch $branch already exists - nothing done." -ForegroundColor Red; return }
if (git ls-remote --heads origin $branch) { Write-Host "Remote branch $branch already exists - nothing done." -ForegroundColor Red; return }
git switch --quiet -c $branch origin/development
if ($LASTEXITCODE -ne 0) { Write-Host "Could not create branch - nothing done." -ForegroundColor Red; return }

$rel = 'dbt/tests/generic/no_deprecated_abbreviations.sql'
$f = Join-Path $repo $rel
$bytes = [IO.File]::ReadAllBytes($f)
$bom = ($bytes.Length -ge 3 -and $bytes[0] -eq 0xEF -and $bytes[1] -eq 0xBB -and $bytes[2] -eq 0xBF)
$t = [IO.File]::ReadAllText($f)
$nl = if ($t.Contains("`r`n")) { "`r`n" } else { "`n" }
if ($t.Contains('flags.WHICH')) { Undo "file already mentions flags.WHICH - stopping"; return }
$rx = '\{%-?\s*if\s+not\s+execute\s*-?%\}'
$hits = [regex]::Matches($t, $rx)
if ($hits.Count -ne 1) {
  Write-Host "Found guard $($hits.Count) time(s). Lines mentioning 'not execute':" -ForegroundColor Yellow
  $i = 0; foreach ($ln in ($t -split "\r?\n")) { $i++; if ($ln -match 'not\s+execute') { "{0,4}: [{1}]" -f $i, $ln } }
  Undo "guard not found exactly once"; return
}
$oldGuard = $hits[0].Value
$newGuard = $oldGuard -replace 'not\s+execute', "not execute or flags.WHICH == 'compile'"
$comment = @'
{# Also skip the registry query under 'dbt compile': dbt_compile runs before dbt_seed,
   so in a fresh or reset environment the seed table does not exist yet and compile
   would deadlock the pipeline. The full check still runs under dbt test / dbt build. #}
'@
$comment = ($comment -replace "\r?\n", $nl)
Write-Host "Old guard: $oldGuard" -ForegroundColor Cyan
Write-Host "New guard: $newGuard" -ForegroundColor Cyan
$t = $t.Replace($oldGuard, $comment + $nl + $newGuard)
[IO.File]::WriteAllText($f, $t, (New-Object Text.UTF8Encoding($bom)))

$changed = @(git diff --name-only)
if (($changed -join '|') -ne $rel) { Undo "expected only $rel to change, got: $($changed -join ', ')"; return }
Write-Host "`n===== DIFF =====" -ForegroundColor Cyan
git --no-pager diff

Push-Location (Join-Path $repo 'dbt')
uv run --no-sync dbt parse --profiles-dir . --target-path target\fix_parse_check
$ok = ($LASTEXITCODE -eq 0)
Pop-Location
if (-not $ok) { Undo "dbt parse failed"; return }
Write-Host "dbt parse: PASSED" -ForegroundColor Green

$msgFile = Join-Path $env:TEMP 'oa_fix_msg.txt'
@'
Fix dbt_compile deadlock: skip abbreviation registry query during compile

no_deprecated_abbreviations ran run_query against seed_abbreviation_standards
whenever execute was true, which includes dbt compile. dbt_compile runs before
dbt_seed, so in any environment where the seed table does not exist yet (DEV
after it went missing, and IVV/PROD on their first run) compile failed and
dbt_seed could never recreate it.

The test now also emits its empty result under 'dbt compile'. The full check
still runs under dbt test / dbt build, after dbt_seed. The ref() outside the
guard is unchanged, so the seed dependency stays in the DAG.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01Q5V13xaQ2dZQoLLE4XZbfJ
'@ | Set-Content -Path $msgFile -Encoding UTF8
git add $rel
git commit --quiet -F $msgFile
if ($LASTEXITCODE -ne 0) { Undo "commit failed (pre-commit hook?) - see output above"; return }
git push -u origin $branch -o ci.skip -o merge_request.create -o merge_request.target=development -o "merge_request.title=Fix dbt_compile deadlock: skip abbreviation registry query during compile"
if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - commit kept locally on $branch; send me the error." -ForegroundColor Red; return }
Write-Host "`nDONE: $(git log -1 --format='%h %s') pushed; MR into development created." -ForegroundColor Green
git switch --quiet $orig
Write-Host "Back on $(git rev-parse --abbrev-ref HEAD)." -ForegroundColor Green
}
