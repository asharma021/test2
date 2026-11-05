Yves, quick update:

Prod extract/load is running. After the ITDS/Tableau change, the prod pipeline finished in about 5 minutes and postgres_extract_load passed.
Mirroring works: 46 of 53 tables mirrored from prod into DEV (about 5.5M rows).
7 tables still fail with column "id" does not exist. They use the shared unique_key: [id] setting but don’t have an id column: message_details, message_thread, message_tracker, notification, bank_account, export_certificates, export_products. Do you know their key columns? Then I can update tables.yml.
Tableau: I’ve asked Ben to open network access from HWPOAAPP05 (our GitLab runner server) to hwptableau02:8060 (the Tableau repository database), since every connection times out right now.
Next: TEST mirror run and checking the mirror schedule inputs.
