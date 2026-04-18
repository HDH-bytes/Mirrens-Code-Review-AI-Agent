# AI Code Review Agent

A Claude-powered code reviewer for GitHub PRs. Listens for webhooks, parses the diff with Python's ast module to pull out the changed functions, asks Claude to review them, and posts inline comments back on the PR.
The trick: Claude's output is locked to a JSON schema via tool_use, and every line number it returns is checked against the function's actual line range before anything hits GitHub. No made-up line numbers. No comments on code that doesn't exist.
What it does

GitHub fires a webhook when a PR opens or gets a new push.
Server verifies the HMAC signature, queues the review, and 200s back fast (GitHub retries anything over ~10s).
Reviewer pulls the diff, figures out which Python functions changed, and sends them to Claude with a strict schema.
Any finding whose line number falls outside the function's actual range gets dropped on the floor.
The rest get posted as inline review comments.
Everything lands in Postgres so we can spot when the same bug pattern keeps showing up in the same function.

Running it
bashcp .env.example .env
# fill in ANTHROPIC_API_KEY, GITHUB_TOKEN, GITHUB_WEBHOOK_SECRET

docker compose up
Postgres spins up with the schema applied. App's on localhost:8000.
Trying it without GitHub
Don't want to set up ngrok just to see it work? There's a local runner that skips the webhook and just prints findings:
bashANTHROPIC_API_KEY=sk-ant-... PYTHONPATH=. \
  python scripts/test_review_locally.py \
  sample_prs/example.py sample_prs/example.diff
The sample file has a SQL injection, a 4-deep nested if, and an eval() on file contents. Claude should flag all three.
Tests
bashpip install -r requirements.txt
PYTHONPATH=. pytest
15 tests. Most of them pin down the AST analyzer: function extraction (top-level, methods, nested, async), signature formatting, complexity math across different branching constructs, line range accuracy, and graceful handling of syntax errors. The rest cover the diff parser.
How it's wired
github webhook
     |
     v
/webhooks/github   (HMAC check, queue background task, 200)
     |
     v
reviewer.review_pr
     |
     +-- fetch PR diff from github
     +-- parse_diff -> changed lines per .py file
     +-- fetch each file at head_sha
     +-- ast_analyzer.analyze -> funcs with line ranges + complexity
     +-- keep only funcs touched by the diff
     +-- claude with submit_review tool (schema enforced)
     +-- drop findings outside func ranges
     +-- post inline comments via github REST
     +-- log everything to postgres
Three tables:

repositories holds per-repo settings (enabled, min severity, disabled categories).
reviews is one row per (repo, PR, head_sha). The unique constraint gives you idempotency on force-push.
review_comments is every inline comment the bot ever posted, indexed on (function_name, category). That index is what powers the "this function has been flagged for the same thing 3 times before" note the bot appends when a pattern repeats.

Wiring up a real webhook

Expose the app publicly. ngrok or a Cloudflare Tunnel is easiest for local testing.
Go to the repo's Settings -> Webhooks and add:

URL: https://<your-host>/webhooks/github
Content type: application/json
Secret: same string as GITHUB_WEBHOOK_SECRET in your .env
Events: Pull requests only


GITHUB_TOKEN needs the repo scope (PAT), or use a GitHub App token with pull_requests: write + contents: read.

The handler only acts on opened, synchronize, and reopened. Pings get a pong.
Per-repo config
First time a PR comes in from a repo, a row is created in repositories with defaults. Tweak those rows to change behavior without redeploying:

Set enabled = false to mute the bot.
Bump min_severity to high if you're tired of low-severity noise.
Add categories to disabled_categories to drop them entirely. I usually toss style in there since linters cover it.

Things I didn't get to

Python only. The ast module is Python-specific. JS/TS would mean swapping in tree-sitter.
FastAPI BackgroundTasks is fine for low volume. If this ever sees real traffic, it wants Celery or RQ with a Redis broker.
The PAT auth path works for one user. Multi-org needs a GitHub App with installation tokens.
McCabe only. Cognitive complexity would be a better signal but I wanted to ship the MVP first.

Security notes

.gitignore excludes .env. Don't commit secrets.
The Postgres creds in docker-compose.yml are local-dev defaults and the port is only bound to localhost. Swap them if you deploy this anywhere.
The webhook handler 401s any request with a bad or missing signature. Don't disable that check.
