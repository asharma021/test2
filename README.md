# Confirm correct branch and clean working tree
if ((git branch --show-current) -ne "pipeline-testing") {
    throw "Stop: not on pipeline-testing"
}

if (git status --porcelain) {
    throw "Stop: working tree is not clean"
}

# Back up the current YAML file
Copy-Item .gitlab-ci.yml .gitlab-ci.yml.backup

# Read YAML
$content = Get-Content .gitlab-ci.yml -Raw

# Identify temporary diagnostic jobs
$jobs = @(
    "check_postgres_test_config",
    "check_el_file_access",
    "check_el_env_loader"
)

# Remove only the named top-level blocks
foreach ($job in $jobs) {
    $pattern = "(?ms)^$([regex]::Escape($job)):\r?\n.*?(?=^[A-Za-z_.][A-Za-z0-9_.-]*:|\z)"

    $matches = [regex]::Matches($content, $pattern)

    if ($matches.Count -ne 1) {
        throw "Expected exactly one block for $job; found $($matches.Count)"
    }

    $content = [regex]::Replace($content, $pattern, "", 1)
}

# Save cleaned YAML
[System.IO.File]::WriteAllText(
    (Join-Path (Get-Location) ".gitlab-ci.yml"),
    $content,
    [System.Text.UTF8Encoding]::new($false)
)

# Review changes
git diff --check
git diff --stat
git status --short
