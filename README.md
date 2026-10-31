Set-Location C:\Users\SharmaAashishR\oa-business-layer
$ErrorActionPreference = 'Stop'
if ((git branch --show-current) -ne 'pipeline-testing') { throw 'STOP: not on pipeline-testing' }
git fetch origin --prune 2>&1 | Out-Null
if ((git rev-parse HEAD) -ne (git rev-parse origin/pipeline-testing)) { throw 'STOP: HEAD differs from origin/pipeline-testing' }
if (git status --porcelain -- .gitlab-ci.yml) { throw 'STOP: .gitlab-ci.yml already has uncommitted changes (press Enter on the old prompt or run: git checkout -- .gitlab-ci.yml)' }
$p = Join-Path $PWD '.gitlab-ci.yml'
$b = [IO.File]::ReadAllBytes($p); $bom = $b.Length -ge 3 -and $b[0] -eq 0xEF -and $b[1] -eq 0xBB
$raw = [IO.File]::ReadAllText($p); $nl = if ($raw.Contains("`r`n")) { "`r`n" } else { "`n" }
$L = [Collections.Generic.List[string]]($raw -split "`r?`n")
$remove = 'postgres_dev_target_inspect', 'postgres_ivv_target_inspect', 'mirror_coverage_inspect', 'check_postgres_test_access'
$removedLines = 0
foreach ($j in $remove) {
  $s = $L.IndexOf("${j}:"); if ($s -lt 0) { throw "STOP: $j not found" }
  $e = $s + 1; while ($e -lt $L.Count -and ($L[$e] -match '^\s' -or $L[$e] -eq '')) { $e++ }
  $c = $s; while ($c -gt 0 -and $L[$c - 1] -match '^#') { $c-- }
  $removedLines += @($L.GetRange($c, $e - $c) | Where-Object { $_ -ne '' }).Count
  $L.RemoveRange($c, $e - $c)
  "Removed $j ($($e - $c) lines)"
}
[IO.File]::WriteAllText($p, ($L -join $nl), (New-Object Text.UTF8Encoding($bom)))
$ErrorActionPreference = 'Continue'
$add = @(git diff -U0 -- .gitlab-ci.yml | Where-Object { $_ -match '^\+[^+]' }).Count
$del = @(git diff -U0 -- .gitlab-ci.yml | Where-Object { $_ -match '^-[^-]' }).Count
$left = @(Select-String -Path $p -Pattern ('^\s*(-\s*)?(job:\s*)?"?(' + ($remove -join '|') + ')"?:?\s*$')).Count
"Changed lines: +$add -$del (expect +0 -$removedLines)   leftover references to removed jobs: $left (expect 0)"
if ($add -eq 0 -and $del -eq $removedLines -and $left -eq 0) {
  git add -- .gitlab-ci.yml
  git commit -q -m "Clean up pipeline-testing: remove temporary diagnostic jobs" -m "Removed postgres_dev_target_inspect, postgres_ivv_target_inspect, mirror_coverage_inspect (hard-coded hosts, diagnostics only) and check_postgres_test_access (obsolete path check). No other job changed."
  git push -o ci.skip origin pipeline-testing
  git log --oneline -2
  "Removed-job count on origin: " + @(git show origin/pipeline-testing:.gitlab-ci.yml | Select-String -Pattern '^(postgres_dev_target_inspect|postgres_ivv_target_inspect|mirror_coverage_inspect|check_postgres_test_access):').Count
  'RESULT: CLEANED UP, COMMITTED AND PUSHED (no pipeline started)'
} else {
  git checkout -- .gitlab-ci.yml
  'RESULT: unexpected change - reverted, nothing committed. Send me this output.'
}
