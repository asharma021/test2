Got it, thanks Josh. So the fix is in tables.yml: the workflow and Permit Mod tables need source: IVV instead of production, and then the existing prod Postgres job loads them from IVV.
