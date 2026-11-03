Hey Yves, every pipeline on a feature branch (Srini’s, Umair’s, mine) shows “stuck” because our only runner (#766) is protected and only picks up jobs from protected branches. Since Josh set this up, I’ll leave the decision to you and Josh. A couple of options:

Quick: add one workflow rule so unprotected branches don’t create pipelines at all. No more “stuck” noise, and checks before merge still come from the local pre-commit hooks (dbt parse, sqlfluff, ruff).
Proper: add a second, unprotected runner under a low-privilege account (no access to our credential files) that runs only the offline checks (lint, dbt parse) on feature and MR pipelines. This needs an admin.

I’d lean toward 1 for now and 2 later, but happy to go with whatever you and Josh prefer. I can make the change once you decide.
