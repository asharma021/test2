Get-ChildItem "\\thqpcelerra01\ttb_common" -Recurse -File -Filter "extract_load.env" -ErrorAction SilentlyContinue |
Select-Object -ExpandProperty FullName

Get-ChildItem "\\thqpcelerra01\ttb_common" -Recurse -Directory -ErrorAction SilentlyContinue |
Where-Object { $_.Name -match "Data_Sharing|dbt_dev_env" } |
Select-Object -ExpandProperty FullName
