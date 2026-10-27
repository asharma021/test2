git grep -n -i -E "oablmsdb|business_layer_test|oa-tst-flexible-server|DBT_PG_DATABASE"


$dir = '\\tthqpcelerra01\ttb_common\_OA_Data_Sharing\OA\dbt_dev_env'

Get-ChildItem -LiteralPath $dir -Filter '*.env' |
  Select-String -Pattern '^(DBT_TARGET|DBT_PG_DATABASE|PGDATABASE|TARGET_PG_DATABASE|SRC_PG_DATABASE(?:_IVV)?)='


  psql -h oa-tst-flexible-server.postgres.database.usgovcloudapi.net -p 5432 -U YOUR_APPROVED_USERNAME -d postgres -c "SELECT datname FROM pg_database WHERE datistemplate = false;"
