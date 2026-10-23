Hi Josh/Michael, I completed the PostgreSQL DEV verify-only pipeline successfully yesterday. Today I added and ran a TEST environment diagnostic on runner #766 (`python-app05`), using the `ttb\svc_python_oa` account.

The diagnostic completed, but the candidate TEST `extract_load.env` path returned NOT ACCESSIBLE.

Can you confirm whether TEST uses the same shared `extract_load.env` file as DEV or has a separate approved file? If separate, could you provide the correct path and confirm that the runner has access?

Once confirmed, I'll configure and run the TEST single-table verify-only job.
