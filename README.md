Test-NetConnection `
  -ComputerName "test-flexible-server-customapps.postgres.database.usgovcloudapi.net" `
  -Port 5432


  @'
import getpass
import psycopg

host = "test-flexible-server-customapps.postgres.database.usgovcloudapi.net"
user = "tst_python_srvc_acct"
password = getpass.getpass("Enter IVV password: ")

try:
    with psycopg.connect(
        host=host,
        port=5432,
        dbname="myttbmsdb",
        user=user,
        password=password,
        sslmode="require",
        connect_timeout=10
    ) as conn:
        print("SUCCESS: IVV PostgreSQL authentication worked")
        with conn.cursor() as cur:
            cur.execute("SELECT current_user, current_database()")
            print(cur.fetchone())
except Exception as e:
    print("FAILED:", str(e))
'@ | uv run --no-sync python -
