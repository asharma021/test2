update on the first prod load:

The production pipeline ran after the merge (pipeline 69708). The Postgres extract/load (job 199869) ran for the full 6 hours and then hit the job timeout.
The main sources loaded into prod with watermarks set:
ITDS: itds_line 52.8M rows, itds_event_details 26.7M, itds_line_hts_list 17.8M+, plus the smaller ITDS tables
myTTB: CBMA claims (cbma_entry_line_item 1.8M, cbma_claims_request_line_item 1.47M, claim_payment, claim_request), chemist cert, export cert, and ref data
ARR: arr_entity, arr_event, company_compliance_detail, the lexis tables, and the rest
All the Tableau tables failed with a connection timeout to hwptableau02 port 8060 (the Tableau Server’s built-in repository database). The runner server HWPOAAPP05 (the Windows server where the GitLab runner executes all pipeline jobs) can’t reach it, so this is a network/firewall issue, not code or credentials. Each Tableau table waited about 2 minutes before failing, and that’s what pushed the job past 6 hours.
The tables that loaded keep their data. Future daily runs will be incremental for them, so they should be much faster.

Two questions:

Should I exclude the Tableau group from the daily prod schedule until PS opens HWPOAAPP05 (GitLab runner Windows server) to hwptableau02:8060 (Tableau repository database)? Otherwise every morning run will lose about an hour to these timeouts and end as failed.
I’ll ask PS to open that network path. Is that the right route, or is there someone specific I should go to? Tableau Server may also need repository access enabled for HWPOAAPP05 by a Tableau admin.

Next, once you confirm, I’ll run the DEV and TEST mirror schedules to validate mirroring from prod.
