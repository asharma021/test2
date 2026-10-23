pool = ConnectionPool(connect_source)
pg_conn = pool.pg()

# Verify-only: compare counts without initializing load-state tables.
if args.verify_only:
    assert count_source is not None
    logger.info(
        msg=f"Verifying {source_name} source vs target counts for {len(mappings)} table(s)"
    )
    comparisons = [
        verify_counts(
            source_conn=pool.source(profile=m.source),
            pg_conn=pg_conn,
            mapping=m,
            count_source=count_source,
        )
        for m in mappings
    ]
    pool.close_all()
    return run_verification(
        comparisons, source_name, args.verify_output
    )

# Initialize load-state tables only for actual loading.
ensure_state_table(pg_conn)
