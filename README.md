Your seed loaded fine, but into business_layer_dev; CI reads business_layer_seed. Quickest temporary fix is to copy it over in DEV (oablmsdb). Please run:

sql
CREATE SCHEMA IF NOT EXISTS business_layer_seed;
CREATE TABLE business_layer_seed.seed_abbreviation_standards AS
  SELECT * FROM business_layer_dev.seed_abbreviation_standards;
GRANT USAGE ON SCHEMA business_layer_seed TO dev_dabl_srvc_acct;
GRANT SELECT, INSERT, UPDATE, DELETE, TRUNCATE
  ON business_layer_seed.seed_abbreviation_standards TO dev_dabl_srvc_acct;
SELECT count(*) FROM business_layer_seed.seed_abbreviation_standards;  -- expect 50

If you have rights, this is even better, so CI owns it like a normal seed:
ALTER TABLE business_layer_seed.seed_abbreviation_standards OWNER TO dev_dabl_srvc_acct;
Then rerun the failed pipeline. dbt_compile should pass, and dbt_seed will refresh the table on its own after that.
