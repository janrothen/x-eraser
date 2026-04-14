Build a Python 3 CLI app called `x_eraser.py` that deletes old posts from an X (Twitter)
account using a downloaded X data archive.

## Requirements

### Platform
- Runs on Raspberry Pi 4 (Linux ARM64, Python 3.13+)
- No GUI, pure CLI

### Commands overview
- x_eraser.py analyze  <archive> [--older-than <threshold>]            → archive analysis, no API calls
- x_eraser.py delete   <archive> --older-than <threshold> [--dry-run]  → delete your own posts
- x_eraser.py unrepost <archive> --older-than <threshold> [--dry-run]  → undo your reposts

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
  - tweet.id_str              → post ID used for DELETE /2/tweets/:id
  - tweet.created_at          → date string, format: "Wed Oct 10 20:19:24 +0000 2018"
  - tweet.full_text           → used to detect replies (starts with @) and for dry-run display
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
- Use OAuth 1.0a via the tweepy library (needed for user-context delete and unrepost)
- The analyze command does not require credentials or a .env file
- If .env is missing or incomplete for delete/unrepost: list which vars are missing and exit

### Pricing
- Define hardcoded constants at the top of the script:
  - COST_PER_DELETE_USD = 0.01   (used by both delete and unrepost commands)
  - COST_PER_READ_USD   = 0.005  (defined for completeness, not used — archive is local)
- Source: X API pay-per-use pricing as of April 2026
  (https://docs.x.com/x-api/getting-started/pricing)
- The unrepost command uses COST_PER_DELETE_USD per call as unrepost is a write operation
  (add a comment in the code to clarify this)
- Read cost is always $0.00 because all data is read from the local archive, not the API
- Add a note in all cost estimates:
  "Prices are hardcoded as of April 2026. Verify current rates at
   https://docs.x.com/x-api/getting-started/pricing"

### Analyze command
- Subcommand: x_eraser.py analyze <archive> [--older-than <threshold>]
- No API calls, no credentials required
- Flags:
  - --older-than <value><unit>  optional, same syntax as delete/unrepost

- Output a summary report:

```
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

- The "With --older-than" block is only shown when --older-than is provided
- Write analysis output to x_eraser.log as well

### Delete command
- Subcommand: x_eraser.py delete <archive> --older-than <threshold>
- Handles two types of entries from data/tweets.js:
  - Original posts (in_reply_to_user_id is empty)
  - Replies (in_reply_to_user_id is present) — only included with --include-replies
  - Reposts are NOT handled here, use the unrepost command instead
- Endpoint: DELETE /2/tweets/:id

- Flags:
  - --older-than <value><unit>  required
  - --dry-run                   list what would be deleted without making API calls
  - --limit <int|all>           how many posts to delete, oldest first (default: all)
  - --include-replies           also delete replies, default: false
  - --delay <float>             seconds between requests (default: 1.0)

- Filter all entries where created_at < (now UTC - threshold)
- Sort filtered entries by created_at ascending (oldest first)
- Apply --limit to the sorted list before deletion

- Rate limit: 50 requests per 15 minutes — no batch delete endpoint exists
- Process sequentially
- Catch HTTP 429, wait for x-rate-limit-reset header + 5s buffer, then retry
- On other API errors: log the error and continue to next post
- After each deleted post, wait --delay seconds

- Pre-run summary:
  - "Found X posts [and Y replies] older than [threshold]."
  - "Deleting [limit] entries (oldest first)."
  - "Replies included: yes/no (use --include-replies to change)"
  - "Estimated API cost: $Y.YY (Z deletes x $0.01)"
  - "Prices are hardcoded as of April 2026. Verify current rates at
     https://docs.x.com/x-api/getting-started/pricing"
  - "Estimated time: ~Xh Ymin at current rate limits (50 deletes / 15 min)."
  - In dry-run: prefix all output with [DRY RUN] and skip time estimate

- Progress output:
  - "[342/1000] Deleted post  1234567890 (2021-03-14): first 60 chars of text..."
  - "[343/1000] Deleted reply 1234567891 (2021-03-15): first 60 chars of text..."

- On completion: "Done. Deleted X posts. Y replies. Z errors."

- Error handling:
  - 404 (already deleted): silently skip, do not count as error
  - KeyboardInterrupt: print summary of what was deleted so far and exit cleanly

### Unrepost command
- Subcommand: x_eraser.py unrepost <archive> --older-than <threshold>
- Endpoint: DELETE /2/users/{id}/retweets/{source_tweet_id}
  (source: https://docs.x.com/x-api/users/unrepost-post)
- Fetch the authenticated user's own ID once at startup via GET /2/users/me
  and reuse it for all unrepost calls as {id}
- Parse reposts from data/retweets.js
- Extract source tweet ID (retweet.retweeted_status.id_str) for each entry

- Flags:
  - --older-than <value><unit>  required
  - --dry-run                   list what would be unreposted without making API calls
  - --limit <int|all>           how many reposts to undo, oldest first (default: all)
  - --delay <float>             seconds between requests (default: 1.0)

- Filter entries where created_at < (now UTC - threshold)
- Sort by created_at ascending (oldest first)
- Apply --limit to the sorted list

- Rate limit: 50 requests per 15 minutes
- Process sequentially
- Catch HTTP 429, wait for x-rate-limit-reset header + 5s buffer, then retry
- On other API errors: log and continue

- Pre-run summary:
  - "Found X reposts older than [threshold]."
  - "Unreposting [limit] reposts (oldest first)."
  - "Estimated API cost: $Y.YY (Z unrepost calls x $0.01)"
  - "Prices are hardcoded as of April 2026. Verify current rates at
     https://docs.x.com/x-api/getting-started/pricing"
  - "Estimated time: ~Xh Ymin at current rate limits (50 requests / 15 min)."
  - In dry-run: prefix all output with [DRY RUN] and skip time estimate

- Progress output:
  - "[12/567] Unreposted source tweet 1346889436626259968 (2021-03-14)"

- On completion: "Done. Unreposted X posts. Y errors."

- Error handling:
  - 404 or retweeted: false response (already unreposted): silently skip, do not count as error
  - KeyboardInterrupt: print summary of what was unreposted so far and exit cleanly

### Logging
- Write a log file x_eraser.log alongside the script with timestamps and all actions
- All three commands write to the same log file
- Log entries must include: timestamp, command, action, post ID, date, result
- On dry-run: log what would have been done, prefixed with [DRY RUN]

### Project structure & tooling
- Package manager: use a venv (python3.13 -m venv .venv)
- Build backend: hatchling
- Project config: pyproject.toml (no setup.py, no setup.cfg)
- Linter: ruff (>=0.15.9)
- Test runner: pytest (~=9.0) with pytest-mock (~=3.15) and pytest-cov (~=7.1)
- Coverage: SonarCloud — generate coverage.xml via pytest-cov with --cov-report=xml
- Minimum test coverage: 80%

### Pinned dependencies
- python-dotenv~=1.2
- tweepy~=4.16.0

### Pinned dev dependencies
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
- Use the official SonarCloud action pinned to a specific commit SHA for security:
  SonarSource/sonarcloud-github-action@eb211723266fe8e83102bac7361f0a05c3ac1d1b
  (do not replace this SHA with @master or a version tag)
- Secrets: GITHUB_TOKEN and SONAR_TOKEN (from GitHub repository secrets)

## Example usage

```bash
# Analyze archive
python3 x_eraser.py analyze <archive>

# Analyze with threshold to preview impact
python3 x_eraser.py analyze <archive> --older-than 3m

# Dry run before deleting
python3 x_eraser.py delete <archive> --older-than 3m --dry-run

# Delete oldest 100 posts
python3 x_eraser.py delete <archive> --older-than 3m --limit 100

# Delete all posts older than 3 months including replies
python3 x_eraser.py delete <archive> --older-than 3m --include-replies

# Dry run before unreposting
python3 x_eraser.py unrepost <archive> --older-than 3m --dry-run

# Unrepost all reposts older than 1 year
python3 x_eraser.py unrepost <archive> --older-than 1y

# Other threshold examples
python3 x_eraser.py delete <archive> --older-than 90d
python3 x_eraser.py delete <archive> --older-than 1y --dry-run
```
