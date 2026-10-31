Set-Location C:\Users\SharmaAashishR\oa-business-layer
$ErrorActionPreference = 'Stop'

if ((git branch --show-current) -ne 'pipeline-testing') { throw 'Not on pipeline-testing - aborting' }
if ((git rev-parse HEAD) -ne (git rev-parse origin/pipeline-testing)) { throw 'HEAD differs from origin/pipeline-testing - aborting' }
if (git status --porcelain -- .gitlab-ci.yml) { throw '.gitlab-ci.yml already has uncommitted changes - aborting' }

function Get-PgBlock([string[]]$L) {
  $s = -1
  for ($i = 0; $i -lt $L.Count; $i++) { if ($L[$i] -match '^postgres_extract_load:\s*$') { if ($s -ge 0) { throw 'Duplicate postgres_extract_load:' }; $s = $i } }
  if ($s -lt 0) { throw 'postgres_extract_load: not found' }
  $e = $s
  for ($i = $s + 1; $i -lt $L.Count; $i++) { if ($L[$i] -match '^\S') { break }; if ($L[$i].Trim() -ne '') { $e = $i } }
  [pscustomobject]@{ Start = $s; End = $e; Lines = [string[]]$L[$s..$e] }
}

$path  = Join-Path $PWD '.gitlab-ci.yml'
$bytes = [IO.File]::ReadAllBytes($path)
$bom   = $bytes.Length -ge 3 -and $bytes[0] -eq 0xEF -and $bytes[1] -eq 0xBB -and $bytes[2] -eq 0xBF
$raw   = [IO.File]::ReadAllText($path)
$nl    = if ($raw.Contains("`r`n")) { "`r`n" } else { "`n" }
$cur   = $raw -split "`r?`n"
$dev   = @(git show "origin/development:.gitlab-ci.yml" | ForEach-Object { $_.TrimEnd("`r") })

$c = Get-PgBlock $cur
$d = Get-PgBlock $dev
$new = @()
if ($c.Start -gt 0) { $new += $cur[0..($c.Start - 1)] }
$new += $d.Lines
if ($c.End -lt $cur.Count - 1) { $new += $cur[($c.End + 1)..($cur.Count - 1)] }

[IO.File]::WriteAllText($path, ($new -join $nl), (New-Object System.Text.UTF8Encoding($bom)))
"Restored postgres_extract_load (was lines $($c.Start+1)-$($c.End+1); now $($d.Lines.Count) lines). Newline style kept: $(if ($nl -eq "`r`n") {'CRLF'} else {'LF'}); BOM: $bom"
