# GPU Watch Bot — Build Order

This document is written to ASD-STE100 Simplified Technical English.

---

## 1. General

Do the phases in sequence. Do not start a phase before the test of the previous phase is satisfactory.

Each phase has three parts:

- Procedure — the steps to do.
- Test — the checks that show the phase is complete.
- Notes — more data about the phase.

---

## 2. Technical names

These names have one meaning in this document:

- **API** — the eBay Browse API.
- **keyset** — the pair of eBay application keys.
- **token** — the client-credentials access token.
- **watchlist** — the list of search terms in the configuration file.
- **search term** — one text query, for example `rx 6600`.
- **listing** — one item for sale on eBay.
- **comp** — a listing in the same result set as the listing under test.
- **trimmed median** — the median price after you remove the highest 10 percent and the lowest 10 percent of the comps.
- **dedup key** — the hash value that identifies the same physical card in more than one listing.
- **poll loop** — the loop that sends one request for each search term, then waits.
- **price snapshot** — one database record of the median price for one search term at one time.
- **fixture file** — the saved API response at `tests/fixtures/search_response.json`.
- **backoff delay** — the time the service waits after an error, before it sends the next request.
- **health endpoint** — the HTTP endpoint at `/healthz`.
- **log record** — one line of structured JSON output.

## 3. Technical verbs

- **to poll** — to send a request to the API at a regular interval.
- **to mint** — to get a new token from the OAuth endpoint.
- **to upsert** — to write a database record, or to change it if it exists.
- **to commit** — to make database changes permanent.
- **to escape** — to change special characters so that the message format does not break.

---

## 4. Phase 0 — Make sure that you can get access to the API

**Time: 1 to 2 hours.**

**CAUTION: DO NOT START PHASE 1 IF THIS TEST FAILS. YOU CAN LOSE MANY HOURS OF WORK.**

### Procedure

1. Register the application. Make a production keyset.
2. Do the compliance procedure for marketplace account deletion notifications.
3. Mint a token with curl. Use the scope `https://api.ebay.com/oauth/api_scope`.
4. Send a search request to the production endpoint. Use the search term `rx 6600`.

### Test

- Make sure that the API gives real data.
- Make sure that the data contains prices and item IDs.
- Write the response to the fixture file.

### Notes

- The keyset stays disabled before you do the compliance procedure.
- Do not use sandbox data. The sandbox has no GPU listings.
- The scope `buy.browse` is not correct. It causes an `invalid_scope` error.

---

## 5. Phase 1 — Make the API client

**Time: 3 to 4 hours.**

### Procedure

1. Write `ebay/client.py`.
2. Add a function that mints a token.
3. Keep the token in memory. Record the expiry time.
4. Mint a new token when the token expires, and after each 401 error.
5. Make a `Listing` data class with these fields: `item_id`, `term`, `title`, `price`, `currency`, `condition`, `url`, `seller`.
6. Write a `search()` function. This function changes the API response into a list of `Listing` objects.

### Test

- Do the unit tests against the fixture file. Do not use the network.
- Make the token expire. Then make sure that the client mints a new token and does not fail.
- Write one test against the live API. Mark this test `integration`, so that it does not run by default.

---

## 6. Phase 2 — Make the database

**Time: 2 to 3 hours.**

### Procedure

1. Write `schema.sql`. Add the tables `listings` and `price_snapshots`.
2. Apply the schema at each start. The operation must be safe if you do it more than one time.
3. Set `PRAGMA journal_mode=WAL`.
4. Write these functions: `upsert_listings()`, `write_snapshot()`, `mark_alerted()`.

### Test

- Do one poll. Then look at the database with the `sqlite3` command. Make sure that the records exist.
- Do the same poll again. Make sure that there are no double records. Make sure that `last_seen` is later than before.

### Notes

- Use the `sqlite3` module. Do not use an ORM. You learn more, and the SQL is visible in the code.

---

## 7. Phase 3 — Remove the double alerts

**Time: 1 to 2 hours.**

### Procedure

1. Write a `normalize()` function for the listing title. Change the text to lower case. Remove the punctuation. Remove the extra spaces.
2. Calculate the dedup key. Use `sha256(normalized_title + seller + round(price / 5))`.
3. Do not send an alert if an alert for the same dedup key occurred in the last 14 days.

### Test

- Make a fixture that contains the same card two times. Give the second listing a different `item_id`, the same seller, the same title, and a price that is 2 dollars different.
- Make sure that the service sends only one alert.
- Make sure that a different card from the same seller still causes an alert.

### Notes

- The `item_id` is not sufficient. If a seller stops a listing and lists the card again, eBay gives a new `item_id` to the same card.

---

## 8. Phase 4 — Find the deals

**Time: 3 to 4 hours.**

### Procedure

1. Write a function that calculates the trimmed median of the comps.
2. Stop the calculation if the number of comps is less than 8. Write a log record. Do not send an alert.
3. Send an alert for each listing with a price below `threshold × median`.
4. Read the `threshold` value for each search term from the configuration file.
5. Write a price snapshot at each poll. Do this if there are deals, and also if there are no deals.

### Test

- This function has no input or output operations. Give it lists of prices. Make sure that the correct listings cause an alert.
- Test a list with an even number of prices. The median calculation is different.
- Test these conditions: no comps, one comp, and seven comps.

### Notes

- Write your own median function and your own percentile function. Do not use the `statistics` module. This is the value of the data structures course.

---

## 9. Phase 5 — Send the Telegram alerts

**Time: 1 to 2 hours.**

### Procedure

1. Use the `sendMessage` method of the Telegram Bot API. Use the MarkdownV2 format.
2. Send one message for each deal. Put this data in the message: title, price, difference from the median, condition, seller, and link.
3. Write a function that escapes the special characters.

### Test

- Send the messages to a private channel. Push an example deal through the full pipeline.
- Test the escape function with a title that contains these characters: `_ * [ ] ( ) ~ > # + - = | { } . !`

### Notes

- MarkdownV2 fails if you do not escape the special characters. This is a frequent error.

---

## 10. Phase 6 — Make the service

**Time: 3 to 4 hours.**

This phase changes the project from a script into a service. Do not remove this phase.

### Procedure

1. Write structured JSON log records to stdout. Write one record for each poll. Put this data in the record: search term, number of comps, median, number of deals, and time.
2. Add a handler for the SIGTERM signal and the SIGINT signal. The handler sets a stop flag.
3. When the stop flag is set: complete the current search term, commit the data, close the database, and stop with exit code 0.
4. Add a backoff delay after each 429 error and each 5xx error. Use `min(base × 2**n, cap) + random.uniform(0, 1)`. Set `n` to zero after a good response.
5. Add the health endpoint on a daemon thread. Give this data: the time of the last good poll, the number of continuous errors, and the database status.
6. Give a non-200 status if the last poll is more than three intervals in the past.
7. Write the systemd unit. Set `Restart=on-failure` and `RestartSec=30`. Put the secrets in an `EnvironmentFile`.

### Test

- Start the unit. Then send SIGTERM during a poll. Make sure that the journal shows a clean stop. Make sure that the database has no incomplete data.
- Use an iptables rule to stop the network access to eBay. Make sure that the service applies the backoff delay. Make sure that the service does not send requests continuously. Make sure that the health endpoint gives a non-200 status.
- Start the machine again. Make sure that the unit starts automatically.
- Send a request to the health endpoint with `curl`. Make sure that the data is correct.

### Notes

- Do not put the secrets in the repository.

---

## 11. Phase 7 — More functions

Do this phase only if you have time.

### Procedure

1. Write a Telegram command handler. Use long polling.
2. Add the command `/history rx6600`. This command shows a sparkline of the data in the `price_snapshots` table.

### Notes

- Do not make the web user interface. This is a different project.

---

## 12. Rules for the sequence

- Make one search term operate through the full pipeline before you use the watchlist. This change is small when the pipeline is correct.
- Do not write an abstraction layer for the retries or for a queue. Use the poll loop, `try`/`except`, a log record, a sleep, and then continue.
- After Phase 0, do all tests against the fixture file. Do not use the live API.

## 13. Time budget

| Phases | Content | Time |
|---|---|---|
| 0 to 5 | The minimum service | 12 to 16 hours |
| 6 | The operational functions | 4 hours |
| 7 | More functions | The remaining time |

## 14. Before you start

Read the requirements for Boot.dev Personal Project 1. Find the necessary deliverables, for example the README format, the test coverage, and the demonstration video. Then you do not have to add these items at hour 28.
