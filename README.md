-- Existing duplicate ids in the candidate table
SELECT count(*) AS row_count,
       count(DISTINCT id) AS distinct_ids,
       count(*) - count(DISTINCT id) AS duplicate_rows
FROM raw_myttb.myttb_controlled_group;

-- Example duplicated ids, if any
SELECT id, count(*) AS copies, min(last_modified_date) AS oldest, max(last_modified_date) AS newest
FROM raw_myttb.myttb_controlled_group
GROUP BY id HAVING count(*) > 1
ORDER BY copies DESC
LIMIT 20;
