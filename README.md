Set-Location C:\Users\SharmaAashishR\oa-business-layer
git fetch origin --prune

"Branch            : $(git branch --show-current)"
"HEAD              : $(git rev-parse --short HEAD)"
"origin/pipe-test  : $(git rev-parse --short origin/pipeline-testing)"
"origin/development: $(git rev-parse --short origin/development)"
"--- Working tree (git status --short) ---"
git status --short
"--- Uncommitted changes to .gitlab-ci.yml (expect none) ---"
git status --porcelain -- .gitlab-ci.yml

function Get-PgBlock([string[]]$L) {
  $s = -1
  for ($i = 0; $i -lt $L.Count; $i++) { if ($L[$i] -match '^postgres_extract_load:\s*$') { if ($s -ge 0) { throw 'Duplicate postgres_extract_load:' }; $s = $i } }
  if ($s -lt 0) { throw 'postgres_extract_load: not found' }
  $e = $s
  for ($i = $s + 1; $i -lt $L.Count; $i++) { if ($L[$i] -match '^\S') { break }; if ($L[$i].Trim() -ne '') { $e = $i } }
  [pscustomobject]@{ Start = $s; End = $e; Lines = [string[]]$L[$s..$e] }
}
$devLines = @(git show "origin/development:.gitlab-ci.yml" | ForEach-Object { $_.TrimEnd("`r") })
$curLines = @([IO.File]::ReadAllLines((Join-Path $PWD '.gitlab-ci.yml')))
$devB = Get-PgBlock $devLines
$curB = Get-PgBlock $curLines
"--- postgres_extract_load: current (lines $($curB.Start+1)-$($curB.End+1)) vs origin/development ---"
Compare-Object $devB.Lines $curB.Lines |
  ForEach-Object { "{0} {1}" -f ($(if ($_.SideIndicator -eq '<=') { 'development:' } else { 'current    :' })), $_.InputObject }
"Identical to development already: $(($devB.Lines -join "`n") -ceq ($curB.Lines -join "`n"))"

# --- OPTIONAL: historical 12/12 banner via API (read-only; needs a token with read_api) ---
if ($env:GITLAB_TOKEN) {
  $GitLab  = 'https://gitlab.ad.ttb.gov'
  $ProjEnc = [uri]::EscapeDataString('office-of-analytics/oa/oa-business-layer')
  $Hdr     = @{ 'PRIVATE-TOKEN' = $env:GITLAB_TOKEN }
  $jobs = foreach ($p in 1..5) {
    Invoke-RestMethod -Uri "$GitLab/api/v4/projects/$ProjEnc/jobs?scope[]=success&per_page=100&page=$p" -Headers $Hdr
  }
  $jobs | Where-Object { $_.ref -eq 'pipeline-testing' -and $_.name -in @('postgres_extract_load', 'postgres_extract_load_test') } |
    ForEach-Object {
      $trace = Invoke-RestMethod -Uri "$GitLab/api/v4/projects/$ProjEnc/jobs/$($_.id)/trace" -Headers $Hdr
      $t = $trace -split "`n"
      [pscustomobject]@{
        Job      = $_.name
        JobId    = $_.id
        Pipeline = $_.pipeline.id
        Sha      = $_.commit.short_id
        Finished = $_.finished_at
        Banner   = (($t | Select-String -Pattern "into '[^']+' database" | Select-Object -First 1).Line -replace '\x1b\[[0-9;]*m', '').Trim()
        IvvSrc   = [bool]($t | Select-String -SimpleMatch 'IVV PostgreSQL source host, port and database selected')
      }
    } | Sort-Object Finished | Format-List
} else {
  "GITLAB_TOKEN not set - get the 12/12 job banner from the GitLab job log (search the log for: into ')"
}
