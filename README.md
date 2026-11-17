DI-4659 — E/L failure notifications → In Progress

Promoted to test and production on 11/17 (MRs development → test → production). Validated on development: a manual pipeline (#71148) ran el_health_check on the runner and passed. In today’s production pipeline (#71213), the E&L jobs now write their status files as expected. The first scheduled 4 AM prod run (11/18) will be the final check: el_health_check runs and fails the pipeline on any unexpected E&L failure. Next: confirm on that run, then follow up with Dyanna on adding BL to the PS pipeline email notifications.

DI-4662 — Env setting integrity check → In Progress

Promoted to test and production on 11/17. In today’s production pipeline (#71213) the env check passed and the E&L jobs ran normally. Will confirm on the 11/18 4 AM run and close.

DI-4663 — Daily E/L health summary → In Progress

The Pages change (publishes el_health_summary.md with the GitLab Pages site) was merged to development on 11/17 and promoted to test and production along with DI-4659 and DI-4662. The el_health_check summary was confirmed on the runner (pipeline #71148). Will confirm the summary on the 11/18 4 AM run. Follow-up: publishing it to Pages from the daily run, since pages doesn’t run on that pipeline today.

DI-4653 — Oracle extract/load to prod → In Progress

Run time against the job limit: done. The project job timeout is 6h, and Oracle’s longest run so far is 2h 14m, so there’s ample headroom. Remaining: review the iris_trade_names_mv row-count difference.
