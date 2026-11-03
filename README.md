& {
$repo = 'C:\Users\SharmaAashishR\oa-business-layer'
Set-Location $repo
$base = 'remote-development'
$branch = 'fix/abbrev-test-compile-remote-dev'
$rel = 'dbt/tests/generic/no_deprecated_abbreviations.sql'
$orig = (git rev-parse --abbrev-ref HEAD).Trim()
function Undo([string]$msg) {
  Write-Host "FAILED: $msg" -ForegroundColor Red
  git cherry-pick --abort 2>$null
  git reset --hard --quiet 2>$null
  git switch --quiet $orig 2>$null
  if ((git rev-parse --abbrev-ref HEAD).Trim() -eq $orig) { git branch -D $branch 2>$null | Out-Null; Write-Host "Reverted: back on '$orig', '$branch' removed. Nothing pushed." -ForegroundColor Yellow }
}
if (git status --porcelain --untracked-files=no) { Write-Host "Uncommitted changes - nothing done." -ForegroundColor Red; return }
git fetch origin --prune 2>$null
if (-not (git ls-remote --heads origin $base)) { Write-Host "origin/$base not found - nothing done." -ForegroundColor Red; return }
if (git branch --list $branch) { Write-Host "Local $branch exists - nothing done." -ForegroundColor Red; return }
if (git ls-remote --heads origin $branch) { Write-Host "Remote $branch exists - nothing done." -ForegroundColor Red; return }
if (git show "origin/${base}:$rel" | Select-String -SimpleMatch "flags.WHICH == 'compile'" -Quiet) { Write-Host "$base already has the fix - nothing to do." -ForegroundColor Yellow; return }
git switch --quiet -c $branch "origin/$base"
if ($LASTEXITCODE -ne 0) { Write-Host "Could not create branch - nothing done." -ForegroundColor Red; return }
git cherry-pick --no-commit 3576e5c
if ($LASTEXITCODE -ne 0) { Undo "conflict applying the fix onto $base - send me this output"; return }
$changed = @(git diff --cached --name-only)
if (($changed -join '|') -ne $rel) { Undo "unexpected files: $($changed -join ', ')"; return }
if (-not (Select-String -Path (Join-Path $repo $rel) -SimpleMatch "flags.WHICH == 'compile'" -Quiet)) { Undo "guard not present after apply"; return }
Write-Host "`n===== DIFF =====" -ForegroundColor Cyan
git --no-pager diff --cached
Push-Location (Join-Path $repo 'dbt')
uv run --no-sync dbt parse --profiles-dir . --target-path target\fix_parse_check_rd
$ok = ($LASTEXITCODE -eq 0)
Pop-Location
if (-not $ok) { Undo "dbt parse failed"; return }
Write-Host "dbt parse: PASSED" -ForegroundColor Green
$msgFile = Join-Path $env:TEMP 'oa_fix_rd_msg.txt'
$msg = "Fix dbt_compile deadlock: skip abbreviation registry query during compile`n`nSame fix as development (3576e5c). no_deprecated_abbreviations queried`nseed_abbreviation_standards during dbt compile, which runs before dbt_seed,`nso compile failed wherever the seed table did not exist yet. The test now`nskips that query under compile; the full check still runs in dbt test / build.`n"
[IO.File]::WriteAllText($msgFile, $msg, (New-Object Text.UTF8Encoding($false)))
git commit --quiet -F $msgFile
if ($LASTEXITCODE -ne 0) { Undo "commit failed (hook?) - see output above"; return }
git push -u origin $branch -o ci.skip -o merge_request.create -o "merge_request.target=$base" -o "merge_request.title=Fix dbt_compile deadlock on remote-development"
if ($LASTEXITCODE -ne 0) { Write-Host "PUSH FAILED - commit kept locally on $branch; send me the error." -ForegroundColor Red; return }
Write-Host "DONE: $(git log -1 --format='%h %s') pushed; MR into $base created." -ForegroundColor Green
git switch --quiet $orig
Write-Host "Back on $(git rev-parse --abbrev-ref HEAD)." -ForegroundColor Green
}
