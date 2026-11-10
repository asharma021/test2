& {
# ===== READ-ONLY: history of vw_fact_sot_fst_returns in oracle_extract_load jobs =====
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
$Base    = 'https://gitlab.ad.ttb.gov/api/v4'
$Proj    = 'office-of-analytics%2Foa%2Foa-business-layer'
$JobName = 'oracle_extract_load'
$View    = 'vw_fact_sot_fst_returns'
$Pages   = 10      # 10 x 100 most recent jobs

$sec = Read-Host 'GitLab token (read_api)' -AsSecureString
$tok = [Runtime.InteropServices.Marshal]::PtrToStringAuto([Runtime.InteropServices.Marshal]::SecureStringToBSTR($sec))
$H   = @{ 'PRIVATE-TOKEN' = $tok }

$jobs = @()
for ($p = 1; $p -le $Pages; $p++) {
  try { $r = Invoke-RestMethod -Headers $H -Uri "$Base/projects/$Proj/jobs?per_page=100&page=$p&scope[]=success&scope[]=failed" }
  catch { Write-Host "API error: $($_.Exception.Message)" -ForegroundColor Red; return }
  if (-not $r) { break }
  $jobs += $r | Where-Object { $_.name -eq $JobName }
}
Write-Host "Found $($jobs.Count) $JobName jobs. Reading logs..." -ForegroundColor Cyan

$rows = foreach ($j in ($jobs | Sort-Object id)) {
  try { $t = (Invoke-WebRequest -UseBasicParsing -Headers $H -Uri "$Base/projects/$Proj/jobs/$($j.id)/trace").Content }
  catch { $t = '' }
  $ok   = [regex]::Match($t, "Done: \S*$View\s*[^\r\n]*")
  $fail = [regex]::Match($t, "$View failed[^\r\n]*")
  $res  = if ($ok.Success) { 'LOADED  ' + ($ok.Value -replace '^Done: \S+\s*-?\s*','') }
          elseif ($fail.Success) { 'FAILED  ' + ($fail.Value -replace "^$View failed\s*",'') }
          else { 'not in this run' }
  [pscustomobject]@{
    Job     = $j.id
    Started = ([datetime]$j.created_at).ToLocalTime().ToString('yyyy-MM-dd HH:mm')
    Ref     = $j.ref
    Status  = $j.status
    View    = $res
  }
}
$rows | Format-Table -AutoSize -Wrap
$tok = $null
}
