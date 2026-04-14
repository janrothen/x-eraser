Build a Python 3 CLI app called `x_eraser.py` that deletes old posts from an X (Twitter)
account using a downloaded X data archive.

## Requirements

### Platform
- Runs on Raspberry Pi 4 (Linux ARM64, Python 3.13+)
- No GUI, pure CLI

### Input
- Positional argument: path to the X archive directory or the tweets.js file directly
- Required flag: --older-than <value><unit> where unit is d (days), w (weeks), m (months),
  y (years). Examples: --older-than 3m, --older-than 90d, --older-than 1y
- Optional flag: --dry-run (list what would be deleted without actually deleting)
- Optional flag: --limit <int|all> how many posts to delete (default: all)
- Optional flag: --delay <float> seconds between requests (default: 1.0)

### X Archive Parsing
- The archive contains a file at data/tweets.js (format: window.YTD.tweets.part0 = [...])
- Strip the JS variable assignment to get valid JSON
- Extract tweet id_str and created_at from each tweet object
- Parse created_at using the format Twitter uses: "Wed Oct 10 20:19:24 +0000 2018"

### Authentication
- Read credentials from a .env file in the same directory as the script
- Required env vars: X_CONSUMER_KEY, X_CONSUMER_SECRET, X_ACCESS_TOKEN, X_ACCESS_TOKEN_SECRET
- Use OAuth 1.0a via the tweepy library (needed for user-context delete)
- Use X API v2 endpoint: DELETE /2/tweets/:id

### Deletion logic
- Filter all tweets where created_at < (now - threshold)
- Sort filtered tweets by date ascending (oldest first)
- Apply --limit to the sorted list before deletion (e.g. --limit 100 deletes the oldest 100)
- DELETE /2/tweets/:id is the only available endpoint — there is no batch delete
- One tweet per request, max 50 requests per 15-minute window (rate limit from X API)
- Process sequentially, respect the 50/15min rate limit
- Catch HTTP 429, wait for x-rate-limit-reset header + 5s buffer, then retry
- On other API errors: log the error and continue to next tweet
- After each deleted tweet, wait --delay seconds (default: 1.0)

### Pricing
- Define a hardcoded constant at the top of the script: COST_PER_DELETE_USD = 0.01
- Define a hardcoded constant: COST_PER_READ_USD = 0.005
- Source: X API pay-per-use pricing as of April 2026 (https://docs.x.com/x-api/getting-started/pricing)
- Before any deletion (and in dry-run), print a cost estimate:
  - Reads: number of posts fetched from archive (no API reads needed — archive is local, so read cost = $0.00)
  - Deletes: count * COST_PER_DELETE_USD
  - Total estimated cost: clearly formatted, e.g. "Estimated API cost: $10.00 (1000 deletes x $0.01)"
  - Add a note: "Prices are hardcoded as of April 2026. Verify current rates at https://docs.x.com/x-api/getting-started/pricing"

### Output / Logging
- Show a summary before starting:
  - "Found X posts older than [threshold]."
  - "Deleting [limit] posts (oldest first)." or "Deleting all X posts (oldest first)."
  - "Estimated API cost: $Y.YY (Z deletes x $0.01)"
  - "Estimated time: ~Xh Ymin at current rate limits (50 deletes / 15 min)."
  - In dry-run: prefix everything with [DRY RUN] and skip the time estimate
- Progress output per deletion: "[342/1000] Deleted tweet 1234567890 (2021-03-14)"
- On completion: "Done. Deleted X posts. Y errors."
- Write a log file x_eraser.log alongside the script with timestamps and all actions
- On dry-run: list each tweet id, date and estimated cost that would be deleted

### Error handling
- If tweets.js is not found: clear error message and exit
- If .env is missing or incomplete: list which vars are missing and exit
- If a tweet was already deleted (404): silently skip, do not count as error
- KeyboardInterrupt: print summary of what was deleted so far and exit cleanly

### Project structure & tooling
- Package manager: use a venv (python3.13 -m venv .venv)
- Build backend: hatchling
- Project config: pyproject.toml (no setup.py, no setup.cfg)
- Linter: ruff
- Test runner: pytest with pytest-mock and pytest-cov
- Coverage: SonarCloud — generate coverage.xml via pytest-cov with --cov-report=xml
- Minimum test coverage: 80%

### Pinned dependencies (use these exact version constraints)
- python-dotenv~=1.2
- tweepy~=4.16.0

### Pinned optional dependencies (use these exact version constraints)
- pytest~=9.0
- pytest-mock~=3.15
- pytest-cov~=7.1
- pre-commit~=4.5
- ruff>=0.15.9

### pyproject.toml requirements
- [build-system] uses hatchling
- [project.scripts] exposes x-eraser as the CLI entry point
- [tool.ruff] configured for Python 3.13+
- [tool.pytest.ini_options] sets --cov=x_eraser --cov-report=xml --cov-report=term-missing
- [tool.coverage.run] excludes tests and __main__ blocks from coverage measurement

### Dependencies (add to requirements.txt)
- tweepy>=4.16.0
- python-dotenv~=1.2

### CI/CD (GitHub Actions)

#### GitGuardian
- Workflow file: .github/workflows/gitguardian.yml
- Trigger: on push and pull_request to main
- Use the official GitGuardian action: gitguardian/ggshield-action@v1
- Secret: GITGUARDIAN_TOKEN (from GitHub repository secrets)

#### SonarCloud
- Workflow file: .github/workflows/sonarcloud.yml
- Trigger: on push and pull_request (opened, synchronize, reopened) to main
- Run pytest with --cov=x_eraser --cov-report=xml before the scan
- Use the official SonarCloud action: SonarSource/sonarcloud-github-action@eb211723266fe8e83102bac7361f0a05c3ac1d1b
- Secrets: GITHUB_TOKEN & SONAR_TOKEN (from GitHub repository secrets)

## Example usage

# Dry run first
python3 x_eraser.py ~/x-archive --older-than 3m --dry-run

# Delete for real
python3 x_eraser.py ~/x-archive --older-than 3m

# Other threshold examples
python3 x_eraser.py ~/x-archive --older-than 90d
python3 x_eraser.py ~/x-archive --older-than 1y --dry-run
