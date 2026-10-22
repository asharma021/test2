Thanks Josh, I reran the GitLab diagnostic and confirmed the runner is using `ttb\svc_python_oa`. The runner can now access `extract_load.env` successfully, although the intermediate directories still show as inaccessible.

I'll proceed with the PostgreSQL DEV verify-only test for `myttb_cg_ref_identifier_type` and review the results.
