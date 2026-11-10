Hi Yves, update on DI-4653 (Oracle extract/load to prod):

This morning’s 4 AM prod run loaded 163 of 164 Oracle tables (145.7M rows, 2h 14m). One table failed: DABL.VW_FACT_SOT_FST_RETURNS, with Oracle error ORA-01841 (invalid year). It’s a bad date value in the source data, not a pipeline issue, so the table will keep failing on every daily run until the source is fixed.
