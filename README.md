Get-ChildItem "src/extract_load" -Recurse -File -Include *.yml,*.yaml,*.json,*.py |
Select-String -Pattern 'myttb_cg_ref_identifier_type' -Context 5,10
