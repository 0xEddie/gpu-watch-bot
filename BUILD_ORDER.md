# GPU Watch Bot — Build Order

## Before you start
Read the Boot.dev Personal Project 1 rubric — README shape, test coverage, demo video

After Phase 0, test against the fixture (not the live API).

## Phase 0 — Register for eBay API access · 1–2h
**Stop here if this fails.**
- Register app, create production keyset
- Complete marketplace account-deletion compliance (keyset stays disabled until then)
- Mint token via curl, scope `https://api.ebay.com/oauth/api_scope` (`buy.browse` → `invalid_scope`)
- `GET item_summary/search?q=rx+6600&limit=50` against production
- **Test:** real prices and item IDs come back → save response to `tests/fixtures/search_response.json`

## Phase 1 — eBay API client · 3–4h
- `ebay/client.py`: mint token, cache in memory with expiry (~2h), re-mint on expiry and on 401
- `Listing` dataclass: `item_id, term, title, price, currency, condition, seller, url`
- `search(term)` flattens the response into `list[Listing]`
- **Test:** unit tests against fixture, no network
- **Test:** force token expiry → confirms re-mint without failing
- **Test:** one live-API test marked `integration`, excluded by default

## Phase 2 — Storage · 2–3h
- `schema.sql` with `listings` + `price_snapshots`; apply idempotently at boot
- `PRAGMA journal_mode=WAL`
- Repo functions: `upsert_listings`, `write_snapshot`, `mark_alerted`, `recent_alerts`
- Raw `sqlite3`, no ORM
- **Test:** one poll → rows visible via `sqlite3` CLI
- **Test:** repeat the poll → no duplicate rows, `last_seen` advanced

## Phase 3 — Dedup · 1–2h
- `normalize(title)`: lowercase, strip punctuation, collapse whitespace
- `dedup_key = sha256(normalized_title + seller + round(price / 5))`
- Suppress if the key was alerted within 14 days
- **Test:** fixture with a relist (new `item_id`, same seller/title, price ±$2) → exactly one alert
- **Test:** a genuinely different card from the same seller still alerts

## Phase 4 — Deal detection · 3–4h
- Own `median` and `percentile`; then `trimmed_median` (drop top/bottom 10%)
- Guard: `n_comps < 8` → log, write snapshot, no alert
- Flag when `price < threshold × median` and `price <= max_price`
- Per-term `threshold` from config
- Write a snapshot every poll, deals or not
- Pure functions, zero I/O
- **Test:** feed price lists, assert which flag
- **Test:** even-length list (median path differs)
- **Test:** 0, 1, and 7 comps

## Phase 5 — Telegram bot set up· 1–2h
- `sendMessage`, MarkdownV2, one message per deal
- Message carries title, price, delta vs median, condition, seller, link
- Escape function for `_ * [ ] ( ) ~ > # + - = | { } . !`
- **Test:** push a synthetic deal through the full pipeline into a private channel
- **Test:** escape function against a title containing every special char

## Phase 6 — Creating a service daemon · 3–4h
- Structured JSON logs to stdout: term, n_comps, median, n_deals, duration
- SIGTERM/SIGINT set a stop flag; loop finishes current term, commits, closes DB, exits 0
- Backoff on 429/5xx: `min(base * 2**n, cap) + random.uniform(0, 1)`, reset `n` on success
- `/healthz` on a daemon thread: last good poll, consecutive errors, DB status; non-200 if last poll > 3 intervals old
- systemd unit: `Restart=on-failure`, `RestartSec=30`, secrets in `EnvironmentFile`
- **Test:** `kill -TERM` mid-poll → clean journal exit, no partial rows
- **Test:** iptables-block eBay → backs off instead of hot-looping, `/healthz` goes non-200
- **Test:** reboot → unit comes back on its own
- **Test:** `curl /healthz` returns accurate values

## Phase 7 — Optional TG bot command
- Telegram long-polling command handler
- `/history rx6600` → sparkline from `price_snapshots`

## Budget
| Phases | | Hours |
|---|---|---|
| 0–5 | MVP | 12–16 |
| 6 | Ops layer | 4 |
| 7 | Stretch | remainder |

