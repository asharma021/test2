& {
Set-Location C:\Users\SharmaAashishR\oa-business-layer
if (Get-Process code -ErrorAction SilentlyContinue) { Write-Host "VS Code is open - it locks .venv. Close ALL VS Code windows, then rerun this block." -ForegroundColor Red; return }
$busy = Get-Process python, pythonw, dbt -ErrorAction SilentlyContinue
if ($busy) { Write-Host "Python/dbt processes running (IDs: $($busy.Id -join ', ')) - close them or reboot, then rerun." -ForegroundColor Red; return }
$bak = Join-Path $env:TEMP ("oa_venv_broken_" + (Get-Date -Format 'yyyyMMddHHmmss'))
if (Test-Path .venv) {
  try { Move-Item .venv $bak -ErrorAction Stop; Write-Host "Old .venv moved to $bak" -ForegroundColor Yellow }
  catch { Write-Host "Could not move .venv (still locked): $($_.Exception.Message)" -ForegroundColor Red; return }
}
uv sync
if ($LASTEXITCODE -ne 0) {
  Write-Host "uv sync FAILED - restoring old .venv" -ForegroundColor Red
  if (Test-Path .venv) { Remove-Item .venv -Recurse -Force -ErrorAction SilentlyContinue }
  if (Test-Path $bak) { Move-Item $bak .venv }
  return
}
uv run --no-sync python -c "import pydantic, pydantic_core, dbt.cli.main; print('pydantic', pydantic.VERSION, '| pydantic-core', pydantic_core.__version__, '| dbt import OK')"
if ($LASTEXITCODE -eq 0) {
  Write-Host "LOCAL .venv REPAIRED. Now rerun the Step 1 block unchanged." -ForegroundColor Green
  Write-Host "(After Step 1 succeeds you can delete $bak)" -ForegroundColor Gray
} else { Write-Host "Imports still failing - send me this output." -ForegroundColor Red }
}
