Select-String -Path src/extract_load/postgres/tables.yml `
  -Pattern 'source:|profile:|myttb_cg_ref_identifier_type|ivv' `
  -Context 3,5
