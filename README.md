Hey Josh, quick update. Oracle, SQL Server and Informix were failing in prod because they weren’t loading the env file, so they had no connection settings. Fixed it and tested on dev in verify-only mode (no loads):

SQL Server: connects, all 10 tables. Ready for prod
Oracle: connects, all 164 tables. Ready for prod
Informix: blocked. The UCCX settings in the env file are still placeholders, and the Informix driver isn’t installed on HWPOAAPP05

Promoting to test and prod now with skip ci so the first full load runs at 4 AM. I’ll check it in the morning.

For Informix, who owns UCCX for the connection details, and who can install the driver on HWPOAAPP05?
