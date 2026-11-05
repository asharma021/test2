Hey Josh, MR !23 (development → test) is ready for your approval. It has:

the prod_mirror env file fix;
the correct key columns for the 7 messaging, banking and exportcert tables;
the CI cleanup (temporary DEV/IVV jobs and tables_ivv_verify.yml removed).

The pipeline is green. After it’s merged, I’ll run the TEST mirror and then promote to production.
