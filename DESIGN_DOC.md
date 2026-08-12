# GPU Watch Bot

Python bot to watch GPU listings on eBay, and send Telegram alerts for good deals.

## Project Scope

- Python service, using eBay Browse API with OAuth client-credentials flow and token refresh
- Polling loop with rate limiting and jittered backoff (API limit: 5k calls/day, 200 items/call)
- Reference a watchlist of search items in a config file
- SQLite to store listings table and price history
- Deal detection triggers based on rolling median or percentile
    - consider using trimmed median
    - min-comps guard
- Listing deduplication - use `normalized_title + seller + price_bucket` not item ID
- Telegram Bot API for sending alerts
- Running on local server with: 
    - structured logging
    -  `/healthz`
    - graceful shutdown
    - and systemd unit 
- Optional: a tiny web UI charting price history per search term 
    - or maybe Telegram command charting price history (per item)

## Architecture
- Single process, single thread for the poll loop; `/healthz` on a daemon thread
- Data flows: `search → normalize → store → score → dedup → notify`
- Every stage below `client` is pure or DB-only, so all of it tests against a saved fixture
```
gpuwatch/
  __init__.py
  config.py      # load + validate watchlist.toml, env secrets
  ebay/
    client.py    # token mint/cache, search(), retry + backoff
    models.py    # Listing dataclass
  store/
    db.py        # connection, schema apply, WAL
    repo.py      # upsert_listings, write_snapshot, mark_alerted, recent_alerts
    schema.sql
  detect.py      # median, percentile, trimmed_median, find_deals  (no I/O)
  dedup.py       # normalize_title, dedup_key, is_suppressed
  notify.py      # Telegram sendMessage, MarkdownV2 escaping
  health.py      # /healthz handler + shared status object
  main.py        # poll loop, signal handlers, wiring
tests/fixtures/search_response.json
deploy/gpuwatch.service
```
 
## Data model
`listings`
- `item_id` TEXT PK, `term` TEXT, `title` TEXT, `price` REAL, `currency` TEXT
- `condition` TEXT, `seller` TEXT, `url` TEXT
- `dedup_key` TEXT, `first_seen` TEXT, `last_seen` TEXT, `alerted_at` TEXT NULL
- index on `dedup_key`, index on `(term, last_seen)`
`price_snapshots`
- `term` TEXT, `ts` TEXT, `n_comps` INTEGER, `median` REAL, `p25` REAL, `min_price` REAL
- one row per term per poll, written whether or not a deal fired

## Config
`watchlist.toml` — checked into the repo, no secrets:
```toml
poll_interval_sec = 300
min_comps = 8
dedup_window_days = 14
 
[[watch]]
term = "rx 6600"
threshold = 0.70      # alert under 70% of trimmed median
max_price = 200
condition = ["USED", "NEW"]
```
Secrets via env / systemd `EnvironmentFile`: `EBAY_CLIENT_ID`, `EBAY_CLIENT_SECRET`, `TELEGRAM_TOKEN`, `TELEGRAM_CHAT_ID`.
 
## Deal detection
- Baseline is cross-sectional, not time-rolling: trimmed median (drop top and bottom 10%) of the current result set for that term
- Skip the term entirely if `n_comps < min_comps`; log and write the snapshot anyway
- Flag a listing when `price < threshold × trimmed_median` and `price <= max_price`
- Own implementations of `median` and `percentile`; handle even-length lists explicitly

## Deduplication
- `dedup_key = sha256(normalize(title) + seller + str(round(price / 5)))`
- `normalize`: lowercase, strip punctuation, collapse whitespace
- Suppress the alert if the same `dedup_key` was alerted within `dedup_window_days`
- Rationale: a seller who ends and relists the same card gets a fresh `item_id`

## Operations
- Structured JSON logs to stdout, one record per term per poll
- Backoff on 429/5xx: `min(base * 2**n, cap) + random.uniform(0, 1)`; reset `n` on success
- `/healthz` returns last successful poll time, consecutive error count, DB reachability; non-200 when the last poll is older than three intervals
- SIGTERM/SIGINT set a stop flag; the loop finishes the current term, commits, closes the DB, exits 0
- systemd: `Restart=on-failure`, `RestartSec=30`, `EnvironmentFile` for secrets

## Known constraints
- Browse returns active listings only — asking prices, not sold prices. The baseline measures the asking distribution, and that is accepted.
- 5,000 API calls/day at the application level. At 300s intervals, ~15 terms stays well inside the cap.
- eBay frames Browse as a buyer-facing surface rather than a research tool, so a limit increase request may be refused. Design assumes no increase

## Architecture
- Single process, single thread for the poll loop; `/healthz` on a daemon thread
- Data flows: `search → normalize → store → score → dedup → notify`
- Every stage below `client` is pure or DB-only, so all of it tests against a saved fixture
```
gpuwatch/
  __init__.py
  config.py      # load + validate watchlist.toml, env secrets
  ebay/
    client.py    # token mint/cache, search(), retry + backoff
    models.py    # Listing dataclass
  store/
    db.py        # connection, schema apply, WAL
    repo.py      # upsert_listings, write_snapshot, mark_alerted, recent_alerts
    schema.sql
  detect.py      # median, percentile, trimmed_median, find_deals  (no I/O)
  dedup.py       # normalize_title, dedup_key, is_suppressed
  notify.py      # Telegram sendMessage, MarkdownV2 escaping
  health.py      # /healthz handler + shared status object
  main.py        # poll loop, signal handlers, wiring
tests/fixtures/search_response.json
deploy/gpuwatch.service
```
 
## Data model
`listings`
- `item_id` TEXT PK, `term` TEXT, `title` TEXT, `price` REAL, `currency` TEXT
- `condition` TEXT, `seller` TEXT, `url` TEXT
- `dedup_key` TEXT, `first_seen` TEXT, `last_seen` TEXT, `alerted_at` TEXT NULL
- index on `dedup_key`, index on `(term, last_seen)`
`price_snapshots`
- `term` TEXT, `ts` TEXT, `n_comps` INTEGER, `median` REAL, `p25` REAL, `min_price` REAL
- one row per term per poll, written whether or not a deal fired

## Config
`watchlist.toml` — checked into the repo, no secrets:
```toml
poll_interval_sec = 300
min_comps = 8
dedup_window_days = 14
 
[[watch]]
term = "rx 6600"
threshold = 0.70      # alert under 70% of trimmed median
max_price = 200
condition = ["USED", "NEW"]
```
Secrets via env / systemd `EnvironmentFile`: `EBAY_CLIENT_ID`, `EBAY_CLIENT_SECRET`, `TELEGRAM_TOKEN`, `TELEGRAM_CHAT_ID`.
 
## Deal detection
- Baseline is cross-sectional, not time-rolling: trimmed median (drop top and bottom 10%) of the current result set for that term
- Skip the term entirely if `n_comps < min_comps`; log and write the snapshot anyway
- Flag a listing when `price < threshold × trimmed_median` and `price <= max_price`
- Own implementations of `median` and `percentile`; handle even-length lists explicitly

## Deduplication
- `dedup_key = sha256(normalize(title) + seller + str(round(price / 5)))`
- `normalize`: lowercase, strip punctuation, collapse whitespace
- Suppress the alert if the same `dedup_key` was alerted within `dedup_window_days`
- Rationale: a seller who ends and relists the same card gets a fresh `item_id`

## Operations
- Structured JSON logs to stdout, one record per term per poll
- Backoff on 429/5xx: `min(base * 2**n, cap) + random.uniform(0, 1)`; reset `n` on success
- `/healthz` returns last successful poll time, consecutive error count, DB reachability; non-200 when the last poll is older than three intervals
- SIGTERM/SIGINT set a stop flag; the loop finishes the current term, commits, closes the DB, exits 0
- systemd: `Restart=on-failure`, `RestartSec=30`, `EnvironmentFile` for secrets

## Known constraints
- Browse returns active listings only — asking prices, not sold prices. The baseline measures the asking distribution, and that is accepted.
- 5,000 API calls/day at the application level. At 300s intervals, ~15 terms stays well inside the cap.
- eBay frames Browse as a buyer-facing surface rather than a research tool, so a limit increase request may be refused. Design assumes no increase

