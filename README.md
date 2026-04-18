# AI Code Review Agent

Agent that listens to GitHub PR webhooks, parses diffs with Python's `ast` module to extract function signatures and complexity, and posts per-function inline review comments via the GitHub REST API. Claude's output is forced through a JSON schema and validated against AST-derived line ranges, so hallucinated line numbers never reach GitHub. Review history and per-repo config live in Postgres.

## How it works

```
GitHub PR webhook
      |
      v
  FastAPI /webhooks/github
      |   (HMAC-SHA256 sig check)
      |   (background task)
      v
  reviewer.review_pr
      |
      +-- fetch_pr_diff
      +-- parse_diff (unidiff) --------> changed lines per .py file
      +-- fetch_file_at_commit
      +-- ast_analyzer.analyze --------> funcs + line ranges + complexity
      +-- functions_touched_by_lines --> only changed funcs
      +-- claude_client.review_functions
      |     - submit_review tool_use schema (forced)
      |     - post-hoc: drop findings whose line is outside the func range
      +-- github_client.post_review_comment
      +-- postgres: repositories, reviews, review_comments
```

The range check in `claude_client.review_functions` is what enforces "no false-positive placements." Schema keeps the shape locked; runtime check keeps the line numbers real.

## Repo layout

```
app/
  main.py                 # FastAPI + webhook + HMAC
  config.py               # pydantic settings
  analyzers/
    ast_analyzer.py       # func extraction + mccabe
    diff_parser.py        # unified diff -> changed lines
  services/
    claude_client.py      # anthropic SDK, tool_use schema, range check
    github_client.py      # PR diff, file at commit, inline comments
    reviewer.py           # orchestrator
  db/
    models.py             # sqlalchemy models
    session.py
    repository.py         # query helpers
  schemas/review.py       # pydantic models
sample_prs/               # a broken file + its diff for local testing
scripts/
  test_review_locally.py  # CLI runner, no github needed
tests/
  test_ast_analyzer.py
  test_diff_parser.py
schema.sql
docker-compose.yml
Dockerfile
requirements.txt
```

## Setup

```bash
cp .env.example .env
# fill in ANTHROPIC_API_KEY, GITHUB_TOKEN, GITHUB_WEBHOOK_SECRET

docker compose up -d db
docker compose up app
```

Postgres schema applies on first startup via `schema.sql`.

## Tests

```bash
pip install -r requirements.txt
PYTHONPATH=. pytest tests/ -v
```

15 tests cover function extraction, signatures, complexity math, line ranges, syntax-error handling, the touched-function filter, and diff parsing.

## Local dry run

`scripts/test_review_locally.py` runs the full pipeline (analysis + Claude + validation) and prints findings instead of posting them.

```bash
ANTHROPIC_API_KEY=sk-ant-... \
  PYTHONPATH=. python scripts/test_review_locally.py \
  sample_prs/example.py sample_prs/example.diff
```

The sample file has a SQL injection, a 4-deep nested if, and an `eval()` of file contents. All three should come back as high-severity findings.

## Wiring up a real webhook

1. Expose the app publicly (ngrok, Cloudflare Tunnel, or a deployment).
2. Repo Settings -> Webhooks:
   - Payload URL: `https://<host>/webhooks/github`
   - Content type: `application/json`
   - Secret: same value as `GITHUB_WEBHOOK_SECRET`
   - Events: "Pull requests" only
3. `GITHUB_TOKEN` needs `repo` scope (PAT) or a GitHub App token with `pull_requests: write` + `contents: read`.

Handler only acts on `opened`, `synchronize`, `reopened`. `ping` returns pong. Everything else returns ignored.

## Per-repo config

Rows in `repositories` are created lazily on first PR:

| column                | effect                                                      |
| --------------------- | ----------------------------------------------------------- |
| `enabled`             | false = mute the bot on this repo                           |
| `min_severity`        | low / medium / high, filters out lower findings             |
| `disabled_categories` | e.g. `{'style'}` to drop style findings entirely            |

## Regression patterns

Every posted comment is persisted with `function_name` and `category`. Before posting a new one, the orchestrator counts prior hits on that pair and, if non-zero, appends a note:

> _Note: `security` issues have been flagged on `get_user_by_email` 3 time(s) before in this repo._

The `(function_name, category)` index on `review_comments` keeps this cheap.

## Security notes

- Never commit `.env`. The `.gitignore` excludes it.
- `docker-compose.yml` uses `postgres:postgres` for local dev only. The port is mapped to `localhost:5433`, not exposed to the network. For any deployment, swap these for real secrets.
- Webhook endpoint rejects unsigned and wrongly-signed requests with 401. Don't disable that check.
- `GITHUB_TOKEN` has write access to PR comments. Store it in your deployment secrets, not in the repo.

## Known limitations

- Python only. AST module is Python-specific. JS/TS would need tree-sitter.
- `BackgroundTasks` is fine for low volume. For real traffic, move to Celery/RQ + Redis.
- PAT path is fine for a single user. Multi-org = GitHub App with installation tokens.
- McCabe only. Cognitive complexity would be a nice addition.
