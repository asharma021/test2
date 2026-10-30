for job in [
    ".postgres_incremental_guard",
    "postgres_extract_load_dev_incremental",
    "postgres_extract_load_ivv_incremental",
]:
    found = any(
        isinstance(d, dict) and job in d
        for d in docs
    )
    print(f"{job}: {'FOUND' if found else 'MISSING'}")
