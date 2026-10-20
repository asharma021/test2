Hi Josh, I’ve configured the PostgreSQL Extract/Load DEV verification job on my feature branch, and the pipeline is being created successfully. However, the job is stuck in pending because runner #766 (`python-app05`) is configured as protected, while my feature branch is unprotected.

Would you recommend enabling an unprotected runner for DEV testing, or is there an approved approach we should use? I don't want to change the shared runner configuration without confirming with you.
