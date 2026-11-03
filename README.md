Hey Yves, your pipeline #69546 is failing on the same seed_abbreviation_standards error, which means your branch doesn’t have the fix from MR !9 yet. Can you merge the latest development into your branch and rerun? That brings in the updated no_deprecated_abbreviations test, and dbt_compile should pass. If your branch has its own changes to that test file, make sure the merge keeps the line {%- if not execute or flags.WHICH == 'compile' -%}.



your branch is missing the fix from MR !9, which is why dbt_compile still fails. To pick it up, from your repo in PowerShell:

powershell
git fetch origin
git switch <your-branch>
git merge origin/development

If Git reports a conflict in dbt/tests/generic/no_deprecated_abbreviations.sql, keep development’s version:

powershell
git checkout --theirs dbt/tests/generic/no_deprecated_abbreviations.sql
git add dbt/tests/generic/no_deprecated_abbreviations.sql
git commit --no-edit

Quick check that the fix is in:

powershell
Select-String -Path dbt\tests\generic\no_deprecated_abbreviations.sql -SimpleMatch "flags.WHICH"

If it prints a line, you’re good. Then push and rerun the pipeline:

powershell
git push

dbt_compile should pass after that.
