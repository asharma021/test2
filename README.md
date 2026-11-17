Publish E&L health summary with GitLab Pages (DI-4663)


## What & why

Per review feedback on DI-4663, the E&L health summary should be available with the GitLab Pages output so all artifacts can be reviewed in one place.

- The `pages` job now runs the existing `scripts/el_health_check.py` after `public/` is built, writing `el_health_summary.md` into the published site (`<pages-root>/<env>/el_health_summary.md`).
- Report-only: the step can never fail the `pages` job.
- `EL_EXPECTED_FAILURES` reuses the `el_health_check` value via `!reference`, so both stay in sync.

CI-only change: 4 lines in the `pages` job. No other jobs, rules, needs or files changed.

## Type of change

- [ ] dbt models / macros / tests / seeds
- [ ] Extract/load (`src/extract_load/`, `tables.yml`)
- [x] CI/CD, tooling, or developer setup
- [ ] Reference Data Update
- [ ] Documentation only

## Data impact

- Models / sources affected: none
- Needs a full refresh after merge? **No**
- Grain, key, or column renames? **No**
- Validation: YAML parsed before/after; only `pages.variables` and `pages.script` changed. CI lint validated in the pipeline editor.

## Notes

- `pages` doesn't currently run on the daily 4 AM prod pipeline, so the summary appears on full builds. Daily runs remain covered by the `el_health_check` artifact and the failed-pipeline alert. Publishing from the daily run is a follow-up.

Refs: DI-4663
