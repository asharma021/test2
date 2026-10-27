test: &pg_test
  type: postgres
  host: oa-tst-flexible-server.postgres.database.usgovcloudapi.net
  port: 5432
  dbname: business_layer_test
  user: "{{ env_var('DBT_PG_USER', '') }}"
  password: "{{ env_var('DBT_PG_PASSWORD', '') }}"
  schema: business_layer_test
  threads: 4
  sslmode: require
