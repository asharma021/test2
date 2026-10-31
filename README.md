-- 1. Does the target table exist?
SELECT to_regclass('raw_myttb.myttb_controlled_group') AS target_exists;

-- 2. Baseline (skip if target_exists is NULL)
SELECT count(*) AS row_count,
       count(DISTINCT id) AS distinct_ids,
       max(last_modified_date) AS max_cursor
FROM raw_myttb.myttb_controlled_group;

-- 3. Existing load-state row (watermark) for this table, if any
SELECT * FROM extract_load_meta._el_state s
WHERE s::text ILIKE '%myttb_controlled_group%';

-- 4. Views that depend on the target table
SELECT DISTINCT v.relnamespace::regnamespace AS view_schema, v.relname AS view_name
FROM pg_depend d
JOIN pg_rewrite r ON r.oid = d.objid
JOIN pg_class v   ON v.oid = r.ev_class
WHERE d.refobjid = to_regclass('raw_myttb.myttb_controlled_group')
  AND v.oid <> d.refobjid;
