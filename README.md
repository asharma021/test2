Continued PostgreSQL Extract/Load IVV verify-only validation.

Resolved the UNC path issue, and the GitLab runner can now successfully access the `extract_load.env` configuration file.

The pipeline progressed to the PostgreSQL TEST database connection but failed with a connection timeout. Additional connectivity testing to the OA TEST PostgreSQL endpoint on port 5432 returned `TcpTestSucceeded: False`.

Currently coordinating with the team to investigate potential network or firewall connectivity issues. Once connectivity is restored, I will rerun the IVV verify-only pipeline and validate the results.
