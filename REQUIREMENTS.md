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
- The archive contains the following relevant files:
  - data/tweets.js    → your own posts and replies
                        format: window.YTD.tweets.part0 = [...]
  - data/retweets.js  → your reposts of other people's posts
                        format: window.YTD.retweet.part0 = [...]

- Parsing approach for both files:
  - Strip the JS variable assignment prefix to get valid JSON
  - The prefix pattern varies per file (e.g. window.YTD.tweets.part0 =,
    window.YTD.retweet.part0 =), so strip everything up to and including
    the first = sign, then parse the remaining JSON array

- From data/tweets.js extract per entry:
  - tweet.id_str        → post ID used for DELETE /2/tweets/:id
  - tweet.created_at    → date string, format: "Wed Oct 10 20:19:24 +0000 2018"
  - tweet.full_text     → used to detect replies (starts with @) and for dry-run display
  - tweet.in_reply_to_user_id → if present and non-empty, the post is a reply

- From data/retweets.js extract per entry:
  - retweet.id_str                  → ID of your repost action
  - retweet.created_at              → date of when you reposted
  - retweet.retweeted_status.id_str → source tweet ID used for
                                      DELETE /2/users/{id}/retweets/{source_tweet_id}

- Classify tweets.js entries into two categories:
  - Reply: in_reply_to_user_id is present and non-empty, OR full_text starts with @
  - Post:  everything else

- If a file is not found, the relevant command should warn the user and skip
  that file gracefully rather than exiting (e.g. if retweets.js is missing,
  unrepost command exits with a clear message; analyze command notes it as
  "retweets.js not found" in the report)

- Parse created_at using the format: "Wed Oct 10 20:19:24 +0000 2018"
  Use datetime.strptime with "%a %b %d %H:%M:%S %z %Y"
  All datetime comparisons must be timezone-aware (UTC)

### Authentication
- Read credentials from a .env file in the same directory as the script
- Required env vars: X_CONSUMER_KEY, X_CONSUMER_SECRET, X_ACCESS_TOKEN, X_ACCESS_TOKEN_SECRET
- Use OAuth 1.0a via the tweepy library (needed for user-context delete)
- Use X API v2 endpoint: DELETE /2/tweets/:id

### Analyze command
- Add a subcommand: x_eraser.py analyze <archive-path>
- No API calls needed, reads only from the local archive files
- No credentials required for this command
- Output a summary report:

```bash
  Archive Analysis
  ----------------
  Posts (original):   1,234
  Reposts:              567
  Replies:              890
  ─────────────────────────
  Total:              2,691

  Oldest post:  2009-11-03
  Newest post:  2026-04-14

  With --older-than 3m:
    Posts to delete:    123
    Reposts to undo:     45
    Replies to delete:   67
    ────────────────────────
    Total affected:     235
    Estimated cost:   $2.35
```

- Optional flag: --older-than (same syntax as delete command) to show
  how many entries would be affected by that threshold
- Parse posts from data/tweets.js
- Parse reposts from data/retweets.js
- Detect replies by checking if tweet text starts with @ or if
  in_reply_to_user_id field is present in the tweet object
- Write analysis output to x_eraser.log as well

### Unrepost command / logic
- Add a subcommand: x_eraser.py unrepost <archive> --older-than <threshold>
- Endpoint: DELETE /2/users/{id}/retweets/{source_tweet_id}
  (source: https://docs.x.com/x-api/users/unrepost-post)
- Parse reposts from data/retweets.js in the archive
- Extract the source tweet ID (the original post that was reposted) from each repost entry
- Filter by --older-than threshold using the repost's created_at date
- Sort by date ascending (oldest first)
- Apply --limit flag (same behaviour as delete command: integer or all, default: all)
- Optional flag: --dry-run (list what would be unreposted without making API calls)
- Optional flag: --delay <float> seconds between requests (default: 1.0)

- Same rate limit as delete: 50 requests per 15 minutes
- Catch HTTP 429, wait for x-rate-limit-reset header + 5s buffer, then retry
- If already unreposted (404 or retweeted: false response): silently skip,
  do not count as error
- On other API errors: log and continue

- Output follows same pattern as delete command:
  - "Found X reposts older than [threshold]."
  - "Unreposting [limit] reposts (oldest first)."
  - "Estimated API cost: $Y.YY (Z unrepost calls x $0.01)"
  - "Estimated time: ~Xh Ymin at current rate limits (50 requests / 15 min)."
  - Progress: "[12/567] Unreposted source tweet 1346889436626259968 (2021-03-14)"
  - On completion: "Done. Unreposted X posts. Y errors."
  - 
### Deletion command / logic
- delete command handles two types of entries from the archive:
  - Original posts (from data/tweets.js where in_reply_to_user_id is empty)
  - Replies (from data/tweets.js where in_reply_to_user_id is present)
  - Both use the same endpoint: DELETE /2/tweets/:id
  - Reposts are NOT handled by the delete command, use the unrepost command instead

- Filter all entries where created_at < (now UTC - threshold)
- Sort filtered entries by created_at ascending (oldest first)
- Apply --limit to the sorted list before deletion
  (e.g. --limit 100 deletes the oldest 100, default: all)

- Optional flag: --include-replies (default: false)
  Without this flag, only original posts are deleted, replies are skipped
  With this flag, both original posts and replies are deleted
  Always clearly indicate in the pre-run summary which types are included

- DELETE /2/tweets/:id is the only available endpoint — there is no batch delete
- One post per request, max 50 requests per 15-minute window
- Process sequentially, respect the 50/15min rate limit
- Catch HTTP 429, wait for x-rate-limit-reset header + 5s buffer, then retry
- On other API errors: log the error and continue to next post
- After each deleted post, wait --delay seconds (default: 1.0)

- Pre-run summary must show:
  - "Found X posts [and Y replies] older than [threshold]."
  - "Deleting [limit] entries (oldest first)."
  - "Replies included: yes/no (use --include-replies to change)"
  - "Estimated API cost: $Y.YY (Z deletes x $0.01)"
  - "Estimated time: ~Xh Ymin at current rate limits (50 deletes / 15 min)."
  - In dry-run: prefix all output with [DRY RUN], skip time estimate

- Progress output per deletion:
  - "[342/1000] Deleted post    1234567890 (2021-03-14): first 60 chars of text..."
  - "[343/1000] Deleted reply   1234567891 (2021-03-15): first 60 chars of text..."

- On completion: "Done. Deleted X posts. Y replies. Z errors."

- Error handling:
  - 404 (already deleted): silently skip, do not count as error
  - KeyboardInterrupt: print summary of what was deleted so far and exit cleanly

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

# Analyze
python3 x_eraser.py analyze <archive>

# Dry run first
python3 x_eraser.py <archive> --older-than <threshold> [--dry-run]

# Delete for real
python3 x_eraser.py <archive> --older-than 3m

# Unrepost
python3 x_eraser.py unrepost <archive> --older-than <threshold> [--dry-run]

# Other threshold examples
python3 x_eraser.py <archive> --older-than 90d
python3 x_eraser.py <archive> --older-than 1y [--dry-run]
