DI-4653 — Oracle extract/load to prod

11/10 4 AM scheduled prod run (pipeline #70387): 163 of 164 tables loaded, 145.7M rows, 2h 14m. One failure: DABL.VW_FACT_SOT_FST_RETURNS with ORA-01841 (invalid year). This is bad date data in the source view, not a pipeline issue. Sent to Ben to fix at the source. This table will keep failing daily until it’s fixed.

DI-4654 — SQL Server extract/load to prod

11/10 4 AM scheduled prod run (pipeline #70387): sqlserver_extract_load completed successfully. Running daily in the prod schedule.

DI-4655 — Informix (UCCX) extract/load to prod

Job is in the daily prod schedule but fails as expected until Ben finishes the source setup. Blocked on source setup.
