# GPU Watch Bot

Python bot to watch GPU listings on eBay and r/hardwareswap, and send Telegram alerts for good deals.

## Project Scope

- Python service
- source listing info (using OAuth client-credentials flow and token refresh) from:
  - eBay Browse API
  - Reddit API (r/hardwareswap)
- Polling loop with rate limiting and jittered backoff
    - eBay: 5k calls/day, 200 items/call
    - Reddit: 100 requests/min per client id
- Normalize r/hardwareswap posts with regex, falling back to a cheap LLM that reads the post body when the title carries no price
- Reference a watchlist of search items in a config file
- SQLite to store listings table and price history
- Deal detection triggers based on rolling median or percentile
    - consider using trimmed median
    - min-comps guard
- Listing deduplication - use `source + normalized_title + seller + price_bucket` not item ID
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
- Data flows: `fetch → normalize → store → score → dedup → notify`
- Both clients emit the same `Listing` type, so every stage below them is source-agnostic
- Every stage below the clients is pure or DB-only, so all of it tests against a saved fixture. The LLM fallback sits *inside* the Reddit client, behind a protocol whose default implementation is a no-op, so the test suite stays offline
```
gpuwatch/
  __init__.py
  config.py      # load + validate watchlist.toml, env secrets
  models.py      # Listing dataclass (shared by both clients)
  ebay/
    client.py    # token mint/cache, search(), retry + backoff
  reddit/
    client.py    # token mint/cache, fetch_new(), retry + backoff
    parse.py     # [H]/[W] tag split, title price extraction  (no I/O)
    llm.py       # Normalizer protocol; LLM fallback reads price from post body
  store/
    db.py        # connection, schema apply, WAL
    repo.py      # upsert_listings, write_snapshot, mark_alerted, recent_alerts
    schema.sql
  detect.py      # median, percentile, trimmed_median, find_deals  (no I/O)
  dedup.py       # normalize_title, dedup_key, is_suppressed
  notify.py      # Telegram sendMessage, MarkdownV2 escaping
  health.py      # /healthz handler + shared status object
  main.py        # poll loop, signal handlers, wiring
tests/fixtures/ebay_search_response.json
tests/fixtures/ebay_sandbox_search.json
tests/fixtures/reddit_new.json
tests/fixtures/reddit_llm_response.json
deploy/gpuwatch.service
```
 
## Data model
`listings`
- `source` TEXT, `item_id` TEXT, PK `(source, item_id)`
- `term` TEXT, `title` TEXT, `price` REAL, `currency` TEXT
- `condition` TEXT, `seller` TEXT, `url` TEXT
- `dedup_key` TEXT, `first_seen` TEXT, `last_seen` TEXT, `alerted_at` TEXT NULL
- `parse_source` TEXT — `regex`, `llm`, or `failed`; records which stage produced the price
- index on `dedup_key`, index on `(term, last_seen)`
- `source` is `ebay` or `reddit`; the PK is composite because a Reddit post id and an eBay item id can collide
`price_snapshots`
- `term` TEXT, `ts` TEXT, `n_comps` INTEGER, `median` REAL, `p25` REAL, `min_price` REAL
- one row per term per poll, written whether or not a deal fired
- the stats are computed from the eBay comps only (see Deal detection), so `n_comps` counts eBay listings

## Config
`watchlist.toml` — checked into the repo, no secrets:
```toml
poll_interval_sec = 300
min_comps = 8
dedup_window_days = 14
ebay_base_url = "https://api.sandbox.ebay.com"   # swap to https://api.ebay.com on the production keyset
subreddit = "hardwareswap"

[llm]
enabled = false                # regex-only until this is turned on
provider = "openrouter"        # or a provider's API directly
model = "..."                  # chosen by JSON reliability, not price
batch_size = 20                # post bodies per request
max_calls_per_poll = 5         # hard ceiling; an outage degrades to regex-only
 
[[watch]]
term = "rx 6600"
threshold = 0.70      # alert under 70% of trimmed median
max_price = 200
condition = ["USED", "NEW"]
```
Secrets via env / systemd `EnvironmentFile`: `EBAY_CLIENT_ID`, `EBAY_CLIENT_SECRET`, `REDDIT_CLIENT_ID`, `REDDIT_CLIENT_SECRET`, `REDDIT_USER_AGENT`, `TELEGRAM_TOKEN`, `TELEGRAM_CHAT_ID`. `OPENROUTER_API_KEY` is required only when `[llm].enabled` is true.
 
## Listing normalization
r/hardwareswap titles follow the `[H]`/`[W]` tag convention, but the asking price usually is not in the title — it sits in the post body, in prose (`$290 shipped CONUS, or $270 local`). Two stages handle this:

1. `reddit/parse.py` — regex only, no I/O. Splits the tags, drops trade-only posts, and looks for a price in the title (`$180`, `180 shipped`, `180 obo`).
2. `reddit/llm.py` — invoked only when stage 1 finds no price. Sends title + selftext to a cheap model and asks for a JSON object of `{price, currency, model, condition}`.

- The fallback is optional. `NullNormalizer` (returns `None`) and `LlmNormalizer` satisfy one `Normalizer` protocol; the null implementation is the default, so the pipeline and the whole test suite run with no network and no API key.
- Each post is sent to the model **once, ever**. `/new` re-serves the same post on every poll for hours; the result is stored on the listing row, keyed by `(source, item_id)`.
- Bodies are batched — up to `batch_size` per request — and capped at `max_calls_per_poll`. An LLM outage degrades to regex-only; it does not stall the poll loop.
- eBay listings never reach this stage. The Browse API returns `price.value` as a number.

### Guarding the model output
A hallucinated price becomes a false alert, so nothing reaches deal detection unvalidated:
- The response must validate against the schema. A malformed response counts as a parse failure, the same as a regex miss.
- `price` must fall inside 10–200% of that term's current trimmed median. Outside the band, drop the listing and count it.
- The extracted model text must match the search term that pulled the post in.
- `parse_source` on the row records `regex`, `llm`, or `failed`, so a bad alert can be traced back to the stage that produced it.

## Deal detection
- Baseline is cross-sectional, not time-rolling: trimmed median (drop top and bottom 10%) of the current result set for that term
- The baseline is computed from the **eBay** comps only — a single r/hardwareswap term rarely returns enough posts to form a median
- Listings from both sources are scored against that one baseline
- Skip the term entirely if `n_comps < min_comps`; log and write the snapshot anyway
- Flag a listing when `price < threshold × trimmed_median` and `price <= max_price`
- Own implementations of `median` and `percentile`; handle even-length lists explicitly

## Deduplication
- `dedup_key = sha256(source + normalize(title) + seller + str(round(price / 5)))`
- `normalize`: lowercase, strip punctuation, collapse whitespace
- Suppress the alert if the same `dedup_key` was alerted within `dedup_window_days`
- Rationale: a seller who ends and relists the same card gets a fresh `item_id`
- `source` is part of the key on purpose: the same card cross-posted to eBay and Reddit is two separate sales, and both are worth an alert

## Operations
- Structured JSON logs to stdout, one record per source per term per poll
- Backoff on 429/5xx: `min(base * 2**n, cap) + random.uniform(0, 1)`; reset `n` on success
- Backoff state is tracked per source, so an eBay outage does not stall the Reddit poll
- `/healthz` returns last successful poll time per source, consecutive error count, DB reachability; non-200 when the last poll is older than three intervals
- SIGTERM/SIGINT set a stop flag; the loop finishes the current term, commits, closes the DB, exits 0
- systemd: `Restart=on-failure`, `RestartSec=30`, `EnvironmentFile` for secrets

## Known constraints
- eBay Browse returns active listings only — asking prices, not sold prices. The baseline measures the asking distribution, and that is accepted.
- 5,000 API calls/day at the application level. At 300s intervals, ~15 terms stays well inside the cap.
- The eBay sandbox holds synthetic inventory with no real GPU listings, so it validates auth and response shape only. Real eBay prices require the production keyset, which is gated behind marketplace account-deletion compliance.
- Reddit's free tier allows 100 requests/min per client id, well above a 300s poll. A generic or default User-Agent is throttled hard, so `REDDIT_USER_AGENT` is required, not optional.
- r/hardwareswap titles are a loose convention, not a schema, and the price is more often in the post body than in the title. Regex handles the tag structure; the LLM fallback reads the body. Posts that neither stage can price are dropped and counted rather than guessed at.
- At a few hundred new posts a day, each parsed once, the LLM fallback costs cents per month. Model choice is therefore driven by structured-output reliability, not by price.
- Reddit post volume per card is low, which is why it contributes listings to score but not comps to the baseline.
