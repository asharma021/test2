Commented out DABL.VW_FACT_SOT_FST_RETURNS in src/extract_load/oracle/tables.yml (BL view, not a source table; its Rcvd_Date logic fails with ORA-01841). Merged to production.

Verified: [date] 4 AM scheduled prod run (pipeline #[number]): oracle_extract_load passed, 163/163 tables loaded, 0 failures.
