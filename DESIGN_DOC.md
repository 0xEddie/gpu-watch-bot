# GPU Watch Bot

Python bot to watch GPU listings on eBay and send Telegram alerts for good deals.

## Project Scope

- Python service
- source listing info (using OAuth client-credentials flow and token refresh) from the eBay Browse API
- Polling loop with rate limiting and jittered backoff
    - eBay: 5k calls/day, 200 items/call
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

### Optional later features

- A Telegram command charting price history (per item)
- Reddit API (r/hardwareswap) as a second source (see [Listing normalization](#listing-normalization-optional-reddit-source))

## Architecture
- Single process, single thread for the poll loop; `/healthz` on a daemon thread
- Data flows: `fetch → normalize → store → score → dedup → notify`
- The client emits a `Listing` type, so every stage below it is source-agnostic. A optional second source added later emits the same type
- Every stage below the client is pure or DB-only, so all of it tests against a saved fixture
```
gpuwatch/
  __init__.py
  config.py      # load + validate watchlist.toml, env secrets
  models.py      # Listing dataclass
  ebay/
    client.py    # token mint/cache, search(), retry + backoff
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
deploy/gpuwatch.service
```

The optional Reddit source, if built, adds `gpuwatch/reddit/` (`client.py`, `parse.py`, `llm.py`) and the fixtures `tests/fixtures/reddit_new.json` and `tests/fixtures/reddit_llm_response.json`. Nothing outside that package changes.
 
## Data model
`listings`
- `source` TEXT, `item_id` TEXT, PK `(source, item_id)`
- `term` TEXT, `title` TEXT, `price` REAL, `currency` TEXT
- `condition` TEXT, `seller` TEXT, `url` TEXT
- `dedup_key` TEXT, `first_seen` TEXT, `last_seen` TEXT, `alerted_at` TEXT NULL
- `parse_source` TEXT — `regex`, `llm`, or `failed`; records which stage produced the price. Always `regex` for eBay, whose prices arrive as numbers
- index on `dedup_key`, index on `(term, last_seen)`
- `source` is `ebay` today. The PK is composite so a second source can be added without a migration — a Reddit post id and an eBay item id can collide
`price_snapshots`
- `term` TEXT, `ts` TEXT, `n_comps` INTEGER, `median` REAL, `p25` REAL, `min_price` REAL
- one row per term per poll, written whether or not a deal fired
- the stats are computed from the eBay comps (see Deal detection), so `n_comps` counts eBay listings

## Config
`watchlist.toml` — checked into the repo, no secrets:
```toml
poll_interval_sec = 300
min_comps = 8
dedup_window_days = 14
ebay_base_url = "https://api.ebay.com"   # https://api.sandbox.ebay.com while the production keyset is pending
 
[[watch]]
term = "rx 6600"
threshold = 0.70      # alert under 70% of trimmed median
max_price = 200
condition = ["USED", "NEW"]
```
Secrets via env / systemd `EnvironmentFile`: `EBAY_CLIENT_ID`, `EBAY_CLIENT_SECRET`, `TELEGRAM_TOKEN`, `TELEGRAM_CHAT_ID`.

The optional Reddit source adds `subreddit`, an `[llm]` table, and the secrets `REDDIT_CLIENT_ID`, `REDDIT_CLIENT_SECRET`, `REDDIT_USER_AGENT`, plus `OPENROUTER_API_KEY` when the LLM fallback is enabled.
 
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
- The baseline is computed from the **eBay** comps — the denser, more uniform pool. A second source added later contributes listings to score, not comps to the baseline
- Skip the term entirely if `n_comps < min_comps`; log and write the snapshot anyway
- Flag a listing when `price < threshold × trimmed_median` and `price <= max_price`
- Own implementations of `median` and `percentile`; handle even-length lists explicitly

## Deduplication
- `dedup_key = sha256(source + normalize(title) + seller + str(round(price / 5)))`
- `normalize`: lowercase, strip punctuation, collapse whitespace
- Suppress the alert if the same `dedup_key` was alerted within `dedup_window_days`
- Rationale: a seller who ends and relists the same card gets a fresh `item_id`
- `source` is part of the key on purpose: once a second source exists, the same card cross-posted to eBay and elsewhere is two separate sales, and both are worth an alert

## Operations
- Structured JSON logs to stdout, one record per source per term per poll
- Backoff on 429/5xx: `min(base * 2**n, cap) + random.uniform(0, 1)`; reset `n` on success
- Backoff state is tracked per source, so one source's outage does not stall another
- `/healthz` returns last successful poll time per source, consecutive error count, DB reachability; non-200 when the last poll is older than three intervals
- SIGTERM/SIGINT set a stop flag; the loop finishes the current term, commits, closes the DB, exits 0
- systemd: `Restart=on-failure`, `RestartSec=30`, `EnvironmentFile` for secrets

## Known constraints
- eBay Browse returns active listings only — asking prices, not sold prices. The baseline measures the asking distribution, and that is accepted.
- 5,000 API calls/day at the application level. At 300s intervals, ~15 terms stays well inside the cap.
- The eBay sandbox holds synthetic inventory with no real GPU listings, so it validates auth and response shape only. Real prices need the production keyset, which is why the keyset is created during account setup rather than late in the build.
- Marketplace account-deletion compliance does not apply: the bot stores listing data only, never seller account data, so the production keyset takes the exemption toggle instead of a callback endpoint.

---

## Listing normalization (optional Reddit source)

r/hardwareswap titles follow the `[H]`/`[W]` tag convention, but the asking price usually is not in the title — it sits in the post body, in prose (`$290 shipped CONUS, or $270 local`). Two stages handle this:

1. `reddit/parse.py` — regex only, no I/O. Splits the tags, drops trade-only posts, and looks for a price in the title (`$180`, `180 shipped`, `180 obo`).
2. `reddit/llm.py` — invoked only when stage 1 finds no price. Sends title + selftext to a cheap model and asks for a JSON object of `{price, currency, model, condition}`.

- The fallback is itself optional. `NullNormalizer` (returns `None`) and `LlmNormalizer` satisfy one `Normalizer` protocol; the null implementation is the default, so the pipeline and the whole test suite run with no network and no API key.
- Each post is sent to the model **once, ever**. `/new` re-serves the same post on every poll for hours; the result is stored on the listing row, keyed by `(source, item_id)`.
- Bodies are batched — up to `batch_size` per request — and capped at `max_calls_per_poll`. An LLM outage degrades to regex-only; it does not stall the poll loop.

### Guarding the model output
A hallucinated price becomes a false alert, so nothing reaches deal detection unvalidated:
- The response must validate against the schema. A malformed response counts as a parse failure, the same as a regex miss.
- `price` must fall inside 10–200% of that term's current trimmed median. Outside the band, drop the listing and count it.
- The extracted model text must match the search term that pulled the post in.
- `parse_source` on the row records `regex`, `llm`, or `failed`, so a bad alert can be traced back to the stage that produced it.

### Reddit constraints
- Reddit's free tier allows 100 requests/min per client id, well above a 300s poll. A generic or default User-Agent is throttled hard, so `REDDIT_USER_AGENT` is required, not optional.
- r/hardwareswap titles are a loose convention, not a schema, and the price is more often in the post body than in the title. Regex handles the tag structure; the LLM fallback reads the body. Posts that neither stage can price are dropped and counted rather than guessed at.
- At a few hundred new posts a day, each parsed once, the LLM fallback costs cents per month. Model choice is therefore driven by structured-output reliability, not by price.
- Reddit post volume per card is low, which is why it would contribute listings to score but not comps to the baseline.
