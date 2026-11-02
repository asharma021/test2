& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
if (Get-Process Code -ErrorAction SilentlyContinue) { Write-Host "Close ALL VS Code windows, then rerun." -ForegroundColor Red; return }
if (Get-Process python, pythonw, dbt -ErrorAction SilentlyContinue) { Write-Host "Close running python/dbt processes (or reboot), then rerun." -ForegroundColor Red; return }
$env:UV_LINK_MODE = 'copy'
Write-Host "--- Cleaning suspect packages from uv cache ---" -ForegroundColor Cyan
uv cache clean tzdata pydantic pydantic-core
$bak = Join-Path $env:TEMP ("oa_venv_broken_" + (Get-Date -Format 'yyyyMMddHHmmss'))
if (Test-Path .venv) {
  try { Move-Item .venv $bak -ErrorAction Stop; Write-Host "Old .venv moved to $bak" -ForegroundColor Yellow }
  catch { Write-Host "Could not move .venv (locked): $($_.Exception.Message)" -ForegroundColor Red; return }
}
$ok = $false
for ($i = 1; $i -le 4; $i++) {
  Write-Host "--- uv sync attempt $i (copy mode) ---" -ForegroundColor Cyan
  $out = @(uv sync 2>&1 | ForEach-Object { "$_" })
  $code = $LASTEXITCODE
  $out | Select-Object -Last 5
  if ($code -eq 0) { $ok = $true; break }
  $m = [regex]::Match(($out -join "`n"), 'Failed to install: ([A-Za-z0-9_.]+?)-\d')
  if (-not $m.Success) { break }
  $pkg = $m.Groups[1].Value
  Write-Host "Clearing uv cache for '$pkg' and retrying..." -ForegroundColor Yellow
  uv cache clean $pkg
}
if (-not $ok) {
  Write-Host "uv sync still FAILING - restoring old .venv. Send me this output." -ForegroundColor Red
  if (Test-Path .venv) { Remove-Item .venv -Recurse -Force -ErrorAction SilentlyContinue }
  if (Test-Path $bak) { Move-Item $bak .venv }
  return
}
uv run --no-sync python -c "import pydantic, pydantic_core, dbt.cli.main; print('pydantic', pydantic.VERSION, '| pydantic-core', pydantic_core.__version__, '| dbt import OK')"
if ($LASTEXITCODE -eq 0) { Write-Host "LOCAL .venv REPAIRED. Now rerun the Step 1 block unchanged." -ForegroundColor Green }
else { Write-Host "Imports still failing - send me this output." -ForegroundColor Red }
}
