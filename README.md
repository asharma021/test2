Promote development to test: exclude ITDS and Tableau from extract/load


## Promote development to test

Brings Josh's extract/load scope change (fda2d1c) to `test`.

### Change
- `src/extract_load/postgres/tables.yml`: ITDS and Tableau table groups commented out.
  - **Tableau:** the runner server HWPOAAPP05 cannot reach the Tableau repository database (hwptableau02:8060); every table timed out after ~2 minutes and pushed the first prod run past the 6-hour job timeout.
  - **ITDS:** excluded from the daily run for now. The first full load into prod is already done (itds_line 52.8M, itds_event_details 26.7M, itds_line_hts_list 17.8M+ rows); the loaded tables stay in prod.

### Still active
- myTTB / ARR groups: imdata, authdata, workflows, messaging, banking, foreignproducer, chemist-cert, exportcert, taxrefdata, ar_mod, claims.
- IVV source groups (permit module tables into `raw_myttb_ivv`).

### Impact
- Config only; no code or pipeline changes.
- Extract/load does not run on `test`.
