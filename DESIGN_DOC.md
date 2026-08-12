# GPU Watch Bot

Python bot to watch GPU listings on eBay, and send Telegram alerts for good deals.

## Project Scope

- Python service, using eBay Browse API with OAuth client-credentials flow and token refresh
- Polling loop with rate limiting and jittered backoff
- Reference a watchlist of search items in a config file
- SQLite to store listings table and price history
- Deal detection triggers based on rolling median or percentile
- Listing deduplication to prevent relisting spam alerts
- Telegram Bot API for sending alerts
- Running on local server with: structured logging, `/healthz`, graceful shutdown, and systemd unit 
- Optional: a tiny web UI charting price history per search term (maybe a Telegram bot command)

