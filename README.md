Skip IVV-only Postgres tables in prod; add inactive Oracle target job (DI-4665)

## What & why

Promote the latest CI/CD and Extract/Load configuration
changes from gh-pull-aashish-dev into development.

The objective is to improve pipeline execution,
environment configuration, and reliability of the
automated Extract/Load process.

## Type of change

- [ ] dbt models / macros / tests / seeds
- [x] Extract/load (`src/extract_load/`, `tables.yml`)
- [x] CI/CD, tooling, or developer setup
- [ ] Reference Data Update
- [ ] Documentation only

## Data impact

- Models / sources affected: Extract/Load pipeline
  configuration and associated environment settings.
- Full refresh after merge: Not anticipated.
- Grain, key, or column renames affecting downstream
  consumers: None anticipated.
- Validation: Development pipeline execution and
  Extract/Load job results to be verified.

## Checklist

- [ ] `uv run pre-commit run` passes.
- [ ] `uv run pyright` passes, if applicable.
- [ ] `uv run bl evaluate` reviewed, if applicable.
- [ ] CI/CD pipeline execution validated.
- [ ] Extract/Load job execution validated.
- [ ] Environment configuration verified.
- [ ] Merge request reviewed and approved.

## Deployment plan

Merge into the development branch and validate pipeline
execution before promoting changes to higher environments.
