Hi Yves, update on the Oracle target job in dev:

Added the DEV TARGET_ORA_* settings as dev-scoped CI/CD variables.
Set ORACLE_CLIENT_BIN_DIR and EL_ORACLE_TEXT_TYPE=VARCHAR2(4000 CHAR) in the job.
Ran it on development (job #208572). The Oracle connection works: ncastestdb901 / PONLTESTDB / dabl[raw_postgres], and thick mode is enabled with the 21.3 client.

The only failure is the source side. The job reads business_layer_mart using the SRC_PG_*_BUSINESS_LAYER settings, which aren't in the EL env file yet, so it fell back to the MyTTB connection and all 7 tables failed with "does not exist" (nothing was written).

Could you let me know the Business Layer source details and where you'd like them set, in the EL env file or as dev CI/CD variables? Then I'll rerun.
