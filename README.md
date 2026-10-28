table_groups:
  - source: IVV
    source_schema: imdata
    target_schema: raw_myttb
    tables:
      - source_table: ref_identifier_type
        target_table: myttb_cg_ref_identifier_type
