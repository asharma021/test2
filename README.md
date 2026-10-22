Hi Michael, I'm not installing a new GitLab runner. I'm using the existing runner #766 (`python-app05`) for the PostgreSQL Extract/Load CI/CD pipeline.

The runner is executing jobs successfully, but it cannot access the shared `_OA_Data_Sharing\OA\dbt_dev_env` directory containing `extract_load.env`.

My Windows account can access the file, but the runner diagnostic shows `NT AUTHORITY\SYSTEM`. I'm trying to confirm whether the existing runner service account has the required network-share and NTFS read permissions.
