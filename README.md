Hey Josh, quick update:

Prod E/L is working. After you commented out ITDS and Tableau, the prod pipeline finished in about 5 minutes and postgres_extract_load passed.
Mirroring works too. prod_mirror wasn’t pointed at the env file, so I fixed that and ran the DEV mirror. 46 of 53 tables came over from prod into DEV (about 5.5M rows): event_log, the CBMA claims tables, ARR, lexis and so on. I had to do a one-time full refresh, since those DEV tables were loaded directly before. Weekly runs should be incremental from here.
7 tables still fail with column "id" does not exist. They use the shared unique_key: [id] setting, but they don’t actually have an id column. They are message_details, message_thread, message_tracker, notification, bank_account, export_certificates and export_products. Do you happen to know their key columns? I’ll update tables.yml once I have them.
The Oracle, SQL Server and Informix tables (IRIS, DABL, UCCX) aren’t in prod yet, so the mirror just skips them for now.
Next up: checking the mirror schedule inputs, running the TEST mirror, and asking for network access from HWPOAAPP05 (GitLab runner server) to hwptableau02:8060 (Tableau repository DB).
