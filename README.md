Your three In Progress tickets are DI-4653, DI-4655 and DI-4662. DI-4659 and DI-4663 are already in Done.

DI-4662 — Env setting integrity check → move to Done

In production since 11/17 and running at the start of every E&L job; it passed for Oracle, SQL Server and Postgres on the daily 4 AM runs. Kept unchanged in the DI-4659 rework (promoting test → production today). Closing.

DI-4653 — Oracle extract/load to prod → keep In Progress

Oracle loads 163/163 tables daily (VW_FACT_SOT_FST_RETURNS removed from config). Run time is 2h 14m against a 6h job limit. As part of the DI-4659 rework (promoting to production today), an Oracle load failure will now fail the pipeline instead of showing as a warning. Remaining: review the iris_trade_names_mv row-count difference, then close.
