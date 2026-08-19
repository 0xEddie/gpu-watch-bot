# GPU Watch Bot — Development Action Plan

Alert bot for underpriced used GPU listings on **eBay** and **r/hardwareswap**, running as a Linux service.

## Before you start
Read the Boot.dev Personal Project 1 rubric — README shape, test coverage, demo video.

After Phase 0, test against fixtures (not the live APIs).

Project architecture and file layout described in `DESIGN_DOC.md`

## Short Version

### Phase 0 — Register for eBay API and Reddit API access · 1–2h active, up to 48h waiting
eBay API use is optional but preferred for sourcing listings.

**eBay developer account**
eBay developer account registration autoblocks free email domains (`@gmail.com`, `@outlook.com`, `@yahoo.com`). Create a developer account using a custom domain or a work address.
- Use a work email, or register a domain (Cloudflare or Porkbun) and use email forwarding
  - Send a test email before registering for eBay developer account — a silently broken forward is indistinguishable from a rejected application
- Register at [developer.ebay.com](https://developer.ebay.com) with `developer@yourdomain.com` or `name@workdomain.com`. Approval arrives by confirmation email, takes 1 business day

**Reddit API**
- Use a Reddit account with a verified email [reddit.com/prefs/apps](https://www.reddit.com/prefs/apps) → **create another app** → type **script** → redirect uri `http://localhost:8080` (required field, unused by a script app)
- Record the client id (the string under the app name) and the secret

- **Test:** sandbox returns a well-formed `itemSummaries` envelope → save to `tests/fixtures/ebay_sandbox_search.json`
- **Note:** sandbox inventory is synthetic and holds no real GPU comps. It proves auth, scope, and response shape only. The real eBay price fixture is captured in Phase 7. Reddit returns real data from day one, so it carries the realistic-data load until then
- **Test:** Reddit returns real r/hardwareswap posts → save to `tests/fixtures/reddit_new.json`

### Phase 1 — eBay API client · 3–4h
- `ebay/client.py`: mint token, cache in memory with expiry (~2h), re-mint on expiry and on 401
- Base URL comes from config, so sandbox → production is a one-line switch in Phase 7
- `Listing` dataclass: `source, item_id, term, title, price, currency, condition, seller, url`
  - `source` is `"ebay"` or `"reddit"`; both clients emit this one type
- `search(term)` flattens the [`item_summary/search`](https://developer.ebay.com/api-docs/buy/browse/resources/item_summary/methods/search) response into `list[Listing]`
- **Test:** unit tests against the fixture, no network
- **Test:** force token expiry → confirms re-mint without failing
- **Test:** one live-API test marked `integration`, excluded by default

### Phase 2 — Reddit client · 3–5h
- `reddit/client.py`: same token cache shape as eBay, ~1h expiry, re-mint on 401
- `fetch_new()` pulls [`/r/hardwareswap/new`](https://www.reddit.com/dev/api#GET_new), paginating on `after` until the page is older than the last poll
- `reddit/parse.py` — **regex only, no I/O.** Parse the title convention: `[USA-CA] [H] RX 6600 [W] PayPal, Local Cash`
  - Keep only the `[H]` (have) side — the `[W]` side is what they want, not what they sell
  - Skip posts whose `[W]` side does not ask for cash/PayPal (those are trades)
  - Extract the price from the title (`$180`, `180 shipped`, `180 obo`)
- `reddit/llm.py` — **LLM fallback**, called only when the title carries no price. On r/hardwareswap the price usually sits in the post body in prose, which regex cannot reach
  - One `Normalizer` protocol, two implementations: `NullNormalizer` (default, returns `None`, keeps every test offline) and `LlmNormalizer`
  - Batch up to 20 bodies per request; cap calls per poll; an outage degrades to regex-only and does not stall the loop
  - Cache the result on the listing row — `/new` re-serves the same post for hours, so each post is parsed once ever
  - Validate before trusting: schema-valid, inside 10–200% of the term median, model text matches the search term. Otherwise drop and count
  - Cheap model via [OpenRouter](https://openrouter.ai/models) or a provider API directly; a few hundred posts/day costs cents per month
- Map onto the same `Listing`: `seller` = author, `url` = permalink, `item_id` = post id, `condition` from flair/title
- Record `parse_source` on the row: `regex`, `llm`, or `failed`
- Drop posts neither stage can price — log and count them, do not crash
- **Test:** fixture covers a clean title, a trade-only post, a multi-item post, a price-in-body post, and an unparseable post
- **Test:** with `NullNormalizer`, the price-in-body post is dropped and the drop counter is non-zero
- **Test:** with a stubbed `LlmNormalizer`, the same post is priced and `parse_source` is `llm`
- **Test:** a stubbed LLM price 10× the median is rejected, not alerted
- **Test:** no exception escapes on any fixture row

### Phase 3 — Storage · 2–3h
- `schema.sql` with `listings` + `price_snapshots`; apply idempotently at boot
- Primary key is `(source, item_id)` — a Reddit post id and an eBay item id can collide
- `PRAGMA journal_mode=WAL`
- Repo functions: `upsert_listings`, `write_snapshot`, `mark_alerted`, `recent_alerts`
- Raw `sqlite3`, no ORM
- **Test:** one poll of both sources → rows for both visible via the `sqlite3` CLI
- **Test:** repeat the poll → no duplicate rows, `last_seen` advanced

### Phase 4 — Dedup · 1–2h
- `normalize(title)`: lowercase, strip punctuation, collapse whitespace
- `dedup_key = sha256(source + normalized_title + seller + round(price / 5))`
- Suppress if the key was alerted within 14 days
- **Test:** fixture with a relist (new `item_id`, same seller/title, price ±$2) → exactly one alert
- **Test:** a genuinely different card from the same seller still alerts
- **Test:** the same card listed on both eBay and Reddit alerts twice — different sources are different sales

### Phase 5 — Deal detection · 3–4h
- Own `median` and `percentile`; then `trimmed_median` (drop top/bottom 10%)
- The baseline comes from the **eBay** comps for that term — the denser, more uniform pool
- Score listings from both sources against that one baseline
- Guard: `n_comps < 8` → log, write snapshot, no alert
- Flag when `price < threshold × median` and `price <= max_price`
- Per-term `threshold` from config
- Write a snapshot every poll, deals or not
- Pure functions, zero I/O
- **Test:** feed price lists, assert which flag
- **Test:** even-length list (median path differs)
- **Test:** 0, 1, and 7 comps

### Phase 6 — Telegram bot setup · 2–3h
- Create Telegram bot using BotFather https://core.telegram.org/bots/features#botfather
- `notify.py`: [`sendMessage`](https://core.telegram.org/bots/api#sendmessage), [MarkdownV2](https://core.telegram.org/bots/api#markdownv2-style), one message per deal
- Message carries source, title, price, delta vs median, condition, seller, link
- Escape function for `_ * [ ] ( ) ~ > # + - = | { } . !`
- Back off on 429 using `parameters.retry_after` from the response body
- **Test:** [`getMe`](https://core.telegram.org/bots/api#getme) returns your bot username — proves the token before any code runs
- **Test:** push a synthetic deal from each source through the full pipeline into a private channel
- **Test:** escape function against a title containing every special char

### Phase 7 — eBay production keyset · 1–2h
Switch to using production keyset before running as a service.
- Create the production keyset in the developer console
- Complete [marketplace account-deletion compliance](https://developer.ebay.com/marketplace-account-deletion) — the keyset stays disabled until this passes
- Point the base URL at `https://api.ebay.com`; the scope string and request shape do not change
- **Test:** `GET item_summary/search?q=rx+6600&limit=50` against production returns real prices and item IDs
- **Test:** save the response to `tests/fixtures/ebay_search_response.json` and re-run the Phase 1 unit tests against it
- **Note:** 5,000 calls/day at the application level ([API call limits](https://developer.ebay.com/develop/apis/api-call-limits)). At 300s intervals, ~15 terms stays well inside the cap

### Phase 8 — Creating a Linux service daemon · 3–4h
- Structured JSON logs to stdout: source, term, n_comps, median, n_deals, duration
- SIGTERM/SIGINT set a stop flag; loop finishes current term, commits, closes DB, exits 0
- Backoff on 429/5xx: `min(base * 2**n, cap) + random.uniform(0, 1)`, reset `n` on success
  - Track backoff per source — an eBay outage must not stall the Reddit poll
- `/healthz` on a daemon thread: last good poll per source, consecutive errors, DB status; non-200 if last poll > 3 intervals old
- systemd unit: `Restart=on-failure`, `RestartSec=30`, secrets in `EnvironmentFile`
- **Test:** `kill -TERM` mid-poll → clean journal exit, no partial rows
- **Test:** iptables-block eBay → backs off instead of hot-looping, Reddit keeps polling, `/healthz` goes non-200
- **Test:** reboot → unit comes back on its own
- **Test:** `curl /healthz` returns accurate values

### Phase 9 — Optional TG bot command
- Telegram long-polling command handler ([`getUpdates`](https://core.telegram.org/bots/api#getupdates))
- `/history rx6600` → sparkline from `price_snapshots`

### Budget
| Phases | | Hours |
|---|---|---|
| 0–6 | MVP | 15–23 |
| 7 | Production keyset | 1–2 |
| 8 | Ops layer | 4 |
| 9 | Stretch | optional |

## Long Version

The detailed, step-by-step form of the phases above. Written to ASD-STE100 Simplified Technical English.

---

### 1. General

Each phase has three parts:

- Procedure — the steps to do.
- Test — the checks that show the phase is complete.
- Notes — more data about the phase.

---

### Technical names

- **keyset** — the pair of eBay application keys.
- **token** — the client-credentials access token.
- **source** — one of the two listing suppliers: eBay or Reddit.
- **script app** — the Reddit application type for a program with no user login.
- **user agent** — the identification text in the HTTP header of each Reddit request.
- **watchlist** — the list of search terms in the configuration file.
- **search term** — one text query, for example `rx 6600`.
- **comp** — a listing in the same result set as the listing under test.
- **trimmed median** — the median price after you remove the highest 10 percent and the lowest 10 percent of the comps.
- **dedup key** — the hash value that identifies the same physical card in more than one listing.
- **poll loop** — the loop that sends one request for each search term, then waits.
- **fixture file** — a saved API response in the directory `tests/fixtures/`.
- **backoff delay** — the time the service waits after an error, before it sends the next request.
- **health endpoint** — the HTTP endpoint at `/healthz`.
- **have tag** — the text `[H]` in an r/hardwareswap title. The text after it is the item for sale.
- **want tag** — the text `[W]` in an r/hardwareswap title. The text after it is the payment the seller accepts.
- **selftext** — the body text of a Reddit post, below the title.
- **normalizer** — the component that reads a price from the post body when the title has no price.
- **BotFather** — the official Telegram bot that makes and configures other Telegram bots.
- **bot token** — the credential that BotFather gives you. It authorizes each request to the Telegram Bot API.
- **chat id** — the number that identifies the Telegram chat, group, or channel that receives the alerts.

### Technical verbs

- **to poll** — to send a request to the API at a regular interval.
- **to mint** — to get a new token from the OAuth endpoint.
- **to upsert** — to write a database record, or to change it if it exists.
- **to commit** — to make database changes permanent.
- **to escape** — to change special characters so that the message format does not break.
- **to forward** — to send email from one address to a different address automatically.
- **to fall back** — to use a second method when the first method gives no result.

---

### Phase 0 — Register for the API access

**Time Estimate: 1 to 2 hours of work. Then wait a maximum of 48 hours for account registration approval.**

**CAUTION: DO NOT START PHASE 1 IF THIS TEST FAILS**

eBay developer account approval takes at least one business day, so start that first.

#### Procedure — eBay developer account signup

1. eBay developer account signup autodenies the free domains (`@gmail.com`, `outlook.com` etc). Get an email address on a custom domain, or use a work email address (skip forward to step 12).
2. Register a domain name. I used Cloudflare Registrar. Porkbun is an alternative, and has first year domain signups for less than 2 USD, with free email forwarding. 
3. Open `Cloudflare dashboard -> Compute -> Email Service -> Email Routing -> Settings`
4. Click through "generate MX records".
5. In `Email Routing -> Destination Addresses`, add a destination address, for example `personalemail@gmail.com`. This is your usual inbox.
6. Open the confirmation email from Cloudflare. Click the link in the message. The route does not operate before this step.
7. In `Email Routing -> Routing rules`, create a new routing rule. Send the mail for `developer@yourdomain.com` to the destination address.
8. Send a test email to `developer@yourdomain.com`. Make sure that the email arrives to your inbox (could land in Spam).
9. Next is to secure your domain email settings. `cloudflare dashboard -> domains -> overview -> select "yournewdomain.com" -> DNS -> Records`
10. There should be an alert above the DNS Records table about creating a DMARC record. Click the alert, and click through generating the `_dmarc` record. This new record should now be displayed at the bottom of the DNS Records table.
11. Next, in `DNS -> Settings`. Find **DNSSEC**. Click Enable.
12. Register for a developer account at [`developer.ebay.com`](https://developer.ebay.com). Use the custom address (again, double check Spam).
13. Wait for the confirmation email. The approval can take up to 48 hours.
14. Create a sandbox keyset.
15. Test the sandbox keyset with curl. Put your two keys in the variables. Then send the commands that follow. The first command mints a token. The second command sends a search request.

```bash
export EBAY_CLIENT_ID='your-sandbox-app-id'
export EBAY_CLIENT_SECRET='your-sandbox-cert-id'

# Mint a token. This scope string is the same on sandbox and production.
EBAY_TOKEN=$(curl -s -X POST https://api.sandbox.ebay.com/identity/v1/oauth2/token \
  -H "Authorization: Basic $(printf '%s:%s' "$EBAY_CLIENT_ID" "$EBAY_CLIENT_SECRET" | base64 -w0)" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d 'grant_type=client_credentials&scope=https://api.ebay.com/oauth/api_scope' \
  | python3 -c 'import json,sys; print(json.load(sys.stdin)["access_token"])')

# Send a search request with that token.
curl -s -G https://api.sandbox.ebay.com/buy/browse/v1/item_summary/search \
  -H "Authorization: Bearer $EBAY_TOKEN" \
  -H "X-EBAY-C-MARKETPLACE-ID: EBAY_US" \
  --data-urlencode 'q=rx 6600' \
  --data-urlencode 'limit=50' \
  | python3 -m json.tool
```

#### Procedure — Reddit API signup

1. Use a Reddit account with a verified email address.
2. Open the page [`reddit.com/prefs/apps`](https://www.reddit.com/prefs/apps). Select **create another app**.
3. Select the type **script**.
4. Put `http://localhost:8080` in the redirect uri field. A script app does not use this value, but the field is necessary.
5. Record the client id. This is the text below the application name.
6. Record the secret.
7. Write a descriptive user agent, for example `gpuwatch/0.1 (by /u/yourname)`.
8. Test the Reddit credentials with curl. Put your two keys and your user agent in the variables. Then send the commands that follow. The first command mints a token. The second command reads the new posts.

```bash
export REDDIT_CLIENT_ID='your-client-id'
export REDDIT_CLIENT_SECRET='your-secret'
export REDDIT_USER_AGENT='gpuwatch/0.1 (by /u/yourname)'

# Mint a token. A script app gets an app-only token from this grant type.
REDDIT_TOKEN=$(curl -s -X POST https://www.reddit.com/api/v1/access_token \
  -H "Authorization: Basic $(printf '%s:%s' "$REDDIT_CLIENT_ID" "$REDDIT_CLIENT_SECRET" | base64 -w0)" \
  -H "User-Agent: $REDDIT_USER_AGENT" \
  -d 'grant_type=client_credentials' \
  | python3 -c 'import json,sys; print(json.load(sys.stdin)["access_token"])')

# Read the new posts. The API host is oauth.reddit.com, not www.reddit.com.
# Send the user agent with every request, including the token request above.
curl -s -G https://oauth.reddit.com/r/hardwareswap/new \
  -H "Authorization: Bearer $REDDIT_TOKEN" \
  -H "User-Agent: $REDDIT_USER_AGENT" \
  --data-urlencode 'limit=100' \
  | python3 -m json.tool
```

#### Test

- Make sure that the eBay sandbox gives a correct `itemSummaries` structure.
- Write the eBay response to the fixture file `tests/fixtures/ebay_sandbox_search.json`.
- Make sure that Reddit gives real posts from r/hardwareswap.
- Write the Reddit response to the fixture file `tests/fixtures/reddit_new.json`.

#### Notes

- The DMARC record prevents email from other persons who use your domain name.
- The DMARC record also decreases the risk that the forwarded email goes to the spam folder.
- DNSSEC prevents changes to your DNS data during transmission.
- The scope `buy.browse` is not correct. It causes an `invalid_scope` error.
- The scope text is the same for the sandbox and for the production environment.
- The sandbox data is artificial. The sandbox has no real GPU listings and no correct prices.
- Thus the sandbox shows only that the authorization and the data structure are correct.
- Reddit gives real data immediately.
- The production keyset is not necessary now. Phase 7 gives the instructions for it.
- Reddit permits 100 requests each minute for each client id. This is sufficient for a 300 second interval.
- Reddit decreases the rate limit for a default user agent or a general user agent. Always send a descriptive user agent.

**Reference**

- eBay developer portal — https://developer.ebay.com
- eBay OAuth client credentials grant — https://developer.ebay.com/api-docs/static/oauth-client-credentials-grant.html
- eBay OAuth scope values — https://developer.ebay.com/api-docs/static/oauth-scopes.html
- eBay Browse API overview — https://developer.ebay.com/api-docs/buy/browse/overview.html
- Reddit application registration — https://www.reddit.com/prefs/apps
- Reddit OAuth2 for script apps — https://github.com/reddit-archive/reddit/wiki/OAuth2
- Reddit API rules, user agent, and rate limits — https://github.com/reddit-archive/reddit/wiki/API

---

### Phase 1 — Make the eBay API client

**Time Estimate: 3 to 4 hours.**

#### Procedure

1. Create `ebay/client.py`.
2. Add a function that mints a token.
3. Keep the token in memory. Record the expiry time.
4. Mint a new token when the token expires, and after each 401 error.
5. Read the base URL from the configuration file. Do not write the base URL in the code.
6. Make a `Listing` data class with these fields: `source`, `item_id`, `term`, `title`, `price`, `currency`, `condition`, `url`, `seller`.
7. Set the `source` field to `ebay`.
8. Write a `search()` function. This function changes the API response into a list of `Listing` objects.

#### Test

- Do the unit tests against the fixture file. Do not use the network.
- Make the token expire. Then make sure that the client mints a new token and does not fail.
- Write one test against the live API. Mark this test `integration`, so that it does not run by default.

#### Notes

- The base URL is a configuration value. Thus Phase 7 changes one line and not the code.
- The two clients give the same `Listing` data class. The subsequent phases do not know the source.

**Reference**

- `item_summary/search` request and response fields — https://developer.ebay.com/api-docs/buy/browse/resources/item_summary/methods/search
- Marketplace ID header values — https://developer.ebay.com/api-docs/static/rest-request-components.html
- eBay OAuth client credentials grant — https://developer.ebay.com/api-docs/static/oauth-client-credentials-grant.html

---

### Phase 2 — Make the Reddit client

**Time Estimate: 3 to 5 hours.**

This phase has two parts. The first part reads the title with a regular expression. The second part reads the post text with a model, but only when the title has no price.

#### Procedure — the client and the title parser

1. Write `reddit/client.py`.
2. Add a function that mints a token. Use the same structure as the eBay client. The token expires after approximately 1 hour.
3. Write a `fetch_new()` function. This function reads `/r/hardwareswap/new`.
4. Read more pages with the `after` parameter. Stop when the posts are older than the last poll.
5. Write `reddit/parse.py`. This module must have no input operations and no output operations.
6. Read the have tag in the title. Keep only the text after `[H]`.
7. Read the want tag in the title. Discard the post if the text after `[W]` does not contain cash or PayPal.
8. Find the price in the title. Accept these formats: `$180`, `180 shipped`, and `180 obo`.

#### Procedure — the LLM fallback

9. Write `reddit/llm.py`. This module finds the price in the post text. On r/hardwareswap the price is more frequently in the post text than in the title. A regular expression cannot read this text correctly.
10. Make a `Normalizer` protocol with one function. The function reads the title and the post text. The function gives a price or `None`.
11. Write two implementations of the protocol. `NullNormalizer` always gives `None`. `LlmNormalizer` sends the text to a model.
12. Use `NullNormalizer` as the default. The service and all the tests must operate with no API key and with no network.
13. Call the normalizer only for the posts that have no price after step 8.
14. Collect these posts. Put a maximum of 20 post texts in one request. Do not send more requests in one poll than the configuration permits.
15. Ask the model for a JSON object with these four fields: `price`, `currency`, `model`, and `condition`.
16. Refuse the result of the model in these three conditions:
    - The structure does not agree with the schema.
    - The price is less than 10 percent of the median for the search term, or more than 200 percent of it.
    - The model text does not agree with the search term.
17. Write the result in the database. Do not send the same post to the model two times. The endpoint `/new` gives the same post at each poll for many hours.
18. If the model or the network fails, use only the result of the title parser. Do not stop the poll loop.

#### Procedure — the listing record

19. Change each post into a `Listing` object. Set the `source` field to `reddit`.
20. Set `seller` to the author name. Set `url` to the permalink. Set `item_id` to the post id.
21. Write `regex`, `llm`, or `failed` in the `parse_source` field of the record.
22. Discard each post that has no price after step 8 and after step 13. Write a log record. Increase a counter.

#### Test

- Make a fixture that contains these posts: a correct post, a trade-only post, a post with more than one item, a post with the price in the post text only, and a post with no price.
- Use `NullNormalizer`. Make sure that the client keeps only the correct post.
- Make sure that the counter of discarded posts is more than zero.
- Use a test implementation of the normalizer. Make sure that the client now finds the price in the post with the price in the post text. Make sure that `parse_source` is `llm`.
- Give the test normalizer a price 10 times more than the median. Make sure that the client refuses this price and does not send an alert.
- Make sure that no exception stops the client.

#### Notes

- A trade-only post is not a sale. The want tag shows the difference.
- Titles from r/hardwareswap are not uniform. Your parser must fail safely.
- The `[H]` and `[W]` structure is regular. The price is not regular. This is the reason for the two parts.
- A model can give an incorrect price. An incorrect price causes an incorrect alert, and an incorrect alert wastes your time. Thus you must always test the result of the model before you use it.
- A few hundred new posts each day is a small quantity. The cost is some cents each month. Select the model for the quality of its JSON output, and not for its price.
- The eBay listings do not use this fallback. The Browse API gives the price as a number.

**Reference**

- Reddit API endpoint list — https://www.reddit.com/dev/api
- The `/new` listing endpoint and the `after` parameter — https://www.reddit.com/dev/api#GET_new
- OpenRouter API documentation — https://openrouter.ai/docs
- OpenRouter model list and prices — https://openrouter.ai/models

---

### Phase 3 — Make the database

**Time Estimate: 2 to 3 hours.**

#### Procedure

1. Write `schema.sql`. Add the tables `listings` and `price_snapshots`.
2. Make the primary key from the two fields `source` and `item_id`.
3. Apply the schema at each start. The operation must be safe if you do it more than one time.
4. Set `PRAGMA journal_mode=WAL`.
5. Write these functions: `upsert_listings()`, `write_snapshot()`, `mark_alerted()`, and `recent_alerts()`.

#### Test

- Do one poll of the two sources. Then look at the database with the `sqlite3` command.
- Make sure that the database contains records from eBay and from Reddit.
- Do the same poll again. Make sure that there are no double records. Make sure that `last_seen` is later than before.

#### Notes

- Use the `sqlite3` module. Do not use an ORM. You learn more, and the SQL is visible in the code.
- A Reddit post id and an eBay item id can be the same text. Thus the primary key has two fields.

---

### Phase 4 — Remove the double alerts

**Time Estimate: 1 to 2 hours.**

#### Procedure

1. Write a `normalize()` function for the listing title. Change the text to lower case. Remove the punctuation. Remove the extra spaces.
2. Calculate the dedup key. Use `sha256(source + normalized_title + seller + round(price / 5))`.
3. Do not send an alert if an alert for the same dedup key occurred in the last 14 days.

#### Test

- Make a fixture that contains the same card two times. Give the second listing a different `item_id`, the same seller, the same title, and a price that is 2 dollars different.
- Make sure that the service sends only one alert.
- Make sure that a different card from the same seller still causes an alert.
- Put the same card on eBay and on Reddit. Make sure that the service sends two alerts.

#### Notes

- The `item_id` is not sufficient. If a seller stops a listing and lists the card again, eBay gives a new `item_id` to the same card.
- The dedup key contains the source. The same card on the two sources is two different sales.

---

### Phase 5 — Find the deals

**Time Estimate: 3 to 4 hours.**

#### Procedure

1. Write a function that calculates the trimmed median of the comps.
2. Use only the eBay comps to calculate the median.
3. Compare the listings from the two sources against this median.
4. Stop the calculation if the number of comps is less than 8. Write a log record. Do not send an alert.
5. Send an alert for each listing with a price below `threshold × median`.
6. Read the `threshold` value for each search term from the configuration file.
7. Write a price snapshot at each poll. Do this if there are deals, and also if there are no deals.

#### Test

- This function has no input or output operations. Give it lists of prices. Make sure that the correct listings cause an alert.
- Test a list with an even number of prices. The median calculation is different.
- Test these conditions: no comps, one comp, and seven comps.

#### Notes

- Write your own median function and your own percentile function. Do not use the `statistics` module. This is the value of the data structures course.
- The eBay result set is larger and more uniform. Thus it gives a better median.
- The number of Reddit posts for one search term is too small for a median.

---

### Phase 6 — Send the Telegram alerts

**Time Estimate: 2 to 3 hours.**

Telegram registration is immediate. There is no approval delay, no compliance procedure, and no custom domain. You do all of it in a chat with a bot.

#### Procedure — register the bot

1. Install Telegram. Use the mobile application, the desktop application, or the web application.
2. Open a chat with [@BotFather](https://t.me/botfather). This is the official Telegram bot that makes other bots.
3. Send the command `/newbot`.
4. Give a display name for the bot, for example `GPU Watch`. You can change this name later.
5. Give a username for the bot. The username must end with `bot`, for example `gpuwatch_alerts_bot`. The username must be unique. You cannot change it later.
6. Record the bot token from the reply. The token has this structure: `123456789:AAExampleTokenTextHere`. This value is `TELEGRAM_TOKEN`.
7. **The bot token is a secret.** A person who has the token has full control of your bot. Do not write the token in the repository. If the token becomes public, send `/revoke` to BotFather. BotFather then makes a new token and stops the old token.
8. Send `/setdescription` and `/setabouttext` to BotFather if you want a description. These steps are not necessary.

#### Procedure — find the chat id

9. The bot cannot start a conversation. You must send a message to the bot first.
   - For a private chat: open the chat with your new bot and send `/start`.
   - For a channel or a group: add the bot to the channel. Then make the bot an administrator with the permission to send messages. Then send one message in the channel.
10. Read the chat id with the `getUpdates` method:

```bash
export TELEGRAM_TOKEN='123456789:AAExampleTokenTextHere'

curl -s "https://api.telegram.org/bot${TELEGRAM_TOKEN}/getUpdates" | python3 -m json.tool
```

11. Find the value `result[].message.chat.id` in the reply. This value is `TELEGRAM_CHAT_ID`. A private chat id is a positive number. A group id or a channel id is a negative number, for example `-1001234567890`.
12. Send a test message:

```bash
export TELEGRAM_CHAT_ID='your-chat-id'

curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage" \
  --data-urlencode "chat_id=${TELEGRAM_CHAT_ID}" \
  --data-urlencode 'text=gpuwatch test message' \
  | python3 -m json.tool
```

The reply contains `"ok": true` if the message is correct.

#### Procedure — write the notifier

13. Write `notify.py`. Use the `sendMessage` method of the Telegram Bot API. Use the MarkdownV2 format.
14. Send one message for each deal. Put this data in the message: source, title, price, difference from the median, condition, seller, and link.
15. Write a function that escapes the special characters.
16. Read `TELEGRAM_TOKEN` and `TELEGRAM_CHAT_ID` from the environment. Do not write these values in the code.
17. Add a backoff delay for the error 429. The reply body contains `parameters.retry_after`. This field gives the number of seconds to wait.

#### Test

- Send the `getMe` method. Make sure that the reply contains the username of your bot. This test shows that the token is correct, before you write any code.
- Send the messages to a private channel. Push an example deal from each source through the full pipeline.
- Test the escape function with a title that contains these characters: `_ * [ ] ( ) ~ > # + - = | { } . !`
- Send a message with an incorrect chat id. Make sure that the service writes a log record and does not stop.

#### Notes

- MarkdownV2 fails if you do not escape the special characters. This is a frequent error. The API gives the message `can't parse entities`.
- The bot cannot send the first message to a person. The person must send `/start` to the bot first. This is a rule of Telegram, and not an error in your code.
- `getUpdates` gives an empty `result` list if the messages are more than 24 hours old. Send a new message and do the request again.
- Do not use `getUpdates` and a webhook at the same time. The two methods are not compatible.
- The limits are approximately 30 messages each second in total, and approximately 20 messages each minute for one group. The alert quantity is much below these limits.

**Reference**

- Telegram Bot API — https://core.telegram.org/bots/api
- BotFather and bot creation — https://core.telegram.org/bots/features#botfather
- `sendMessage` — https://core.telegram.org/bots/api#sendmessage
- MarkdownV2 format and the characters to escape — https://core.telegram.org/bots/api#markdownv2-style
- `getUpdates` — https://core.telegram.org/bots/api#getupdates
- `getMe` — https://core.telegram.org/bots/api#getme

---

### Phase 7 — Make the eBay production keyset

**Time Estimate: 1 to 2 hours.**

The sandbox keyset is sufficient for Phase 1 to Phase 6. Change to the production keyset before you run the service.

#### Procedure

1. Make a production keyset in the developer console.
2. Do the [compliance procedure for marketplace account deletion notifications](https://developer.ebay.com/marketplace-account-deletion).
3. Make a public HTTPS endpoint. This endpoint sends the challenge code from eBay back to eBay. Use the domain from Phase 0.
4. Change the base URL in the configuration file to `https://api.ebay.com`.
5. Send a test search request to the production endpoint. Use the search term `rx 6600`.

#### Test

- Make sure that the API gives real prices and real item IDs.
- Write the response to the fixture file `tests/fixtures/ebay_search_response.json`.
- Run the Phase 1 unit tests again against this fixture file. Do not change the client code.

#### Notes

- The production keyset stays disabled before the compliance procedure is complete.
- The scope text and the request structure do not change between the two environments.
- The limit is 5000 calls each day for each application. At an interval of 300 seconds, 15 search terms stay below the limit.

**Reference**

- Marketplace account deletion notification requirements — https://developer.ebay.com/marketplace-account-deletion
- eBay API call limits — https://developer.ebay.com/develop/apis/api-call-limits
- Your application keysets (sandbox and production) — https://developer.ebay.com/my/keys

---

### Phase 8 — Make the service

**Time Estimate: 3 to 4 hours.**

This phase changes the project from a script into a service.

#### Procedure

1. Write structured JSON log records to stdout. Write one record for each poll. Put this data in the record: source, search term, number of comps, median, number of deals, and time.
2. Add a handler for the SIGTERM signal and the SIGINT signal. The handler sets a stop flag.
3. When the stop flag is set: complete the current search term, commit the data, close the database, and stop with exit code 0.
4. Add a backoff delay after each 429 error and each 5xx error. Use `min(base × 2**n, cap) + random.uniform(0, 1)`. Set `n` to zero after a good response.
5. Keep a different backoff counter for each source.
6. Add the health endpoint on a daemon thread. Give this data: the time of the last good poll for each source, the number of continuous errors, and the database status.
7. Give a non-200 status if the last poll is more than three intervals in the past.
8. Write the systemd unit. Set `Restart=on-failure` and `RestartSec=30`. Put the secrets in an `EnvironmentFile`.

#### Test

- Start the unit. Then send SIGTERM during a poll. Make sure that the journal shows a clean stop. Make sure that the database has no incomplete data.
- Use an iptables rule to stop the network access to eBay. Make sure that the service applies the backoff delay. Make sure that the service does not send requests continuously.
- Make sure that the Reddit poll continues during the eBay failure.
- Make sure that the health endpoint gives a non-200 status.
- Start the machine again. Make sure that the unit starts automatically.
- Send a request to the health endpoint with `curl`. Make sure that the data is correct.

#### Notes

- Do not put the secrets in the repository.
- A failure of one source must not stop the other source. Thus each source has its own backoff counter.

---

### Phase 9 — More functions

Do this phase only if you have time.

#### Procedure

1. Write a Telegram command handler. Use long polling with the `getUpdates` method.
2. Register the command with BotFather. Send `/setcommands`, then give the command list.
3. Add the command `/history rx6600`. This command shows a sparkline of the data in the `price_snapshots` table.

**Reference**

- `getUpdates` and the `offset` parameter for long polling — https://core.telegram.org/bots/api#getupdates
- Bot commands and `/setcommands` — https://core.telegram.org/bots/features#commands

---

### Rules for the sequence

- Make one search term operate through the full pipeline before you use the watchlist. This change is small when the pipeline is correct.
- Do not write an abstraction layer for the retries or for a queue. Use the poll loop, `try`/`except`, a log record, a sleep, and then continue.
- After Phase 0, do all tests against the fixture files. Do not use the live APIs.
- Add the second source only when the first source operates through the full pipeline.

