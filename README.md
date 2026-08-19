# GPU Watch Bot

An alert bot for cheap used GPU listings. It polls **eBay** (Browse API) and **r/hardwareswap** (Reddit API), compares each listing against a rolling price baseline for that card, and sends a Telegram message when something is priced well below the going rate.

This runs as a Linux service (systemd).

---

## How to run it

<!-- TODO: installation -->

<!-- TODO: configuration — watchlist.toml, env secrets -->

<!-- TODO: running locally -->

<!-- TODO: installing the systemd unit -->

### Registering for API access

Both sources need credentials before the bot can run. eBay developer account approval can take up to 48 hours.

#### Custom domain email (Cloudflare)

eBay's developer registration rejects free email providers(`@gmail.com`, `@outlook.com`, `@yahoo.com`), so you need an address on a domain you control. I used Cloudflare Registrar. You could use a work email. Porkbun also works; they have promos for several cheap TLD (first year on `.sgd` is below $2USD). Porkbun's email forwarding skips most of the steps to set up email forwarding.

**1. Route mail to your real inbox**

Cloudflare dashboard → **Compute** → **Email Service** → **Email Routing** → **Settings**.

- Let Cloudflare **autogenerate the MX records**. It writes the MX and SPF entries into your.
- Add a **destination address** — your normal email inbox, e.g. `personalemail@gmail.com`. Cloudflare emails it a confirmation link; click that link before going further, or the route stays inactive.
- Add a **routing rule**: mail to `developer@yournewdomain.com` forwards to `personalemail@gmail.com`.

Send yourself a test email before registering with eBay to verify that email forwarding works.

**2. Stop others from sending mail posing as your domain (DMARC)**

Cloudflare dashboard → **Domains** → **Overview** → select `yournewdomain.com` → **DNS** → **Records**.

Cloudflare shows an alert above the DNS Records table, suggesting you **create a DMARC record**. Click through it, let it generate the `_dmarc` entry, and add that to your DNS records.

Without DMARC, anyone can send mail claiming to be from your domain. It also improves the odds that forwarded mail lands in the inbox instead of spam.

**3. Enable DNSSEC**

**DNS** → **Settings** → find **DNSSEC** click **Enable**.

DNSSEC signs your DNS responses so they cannot be tampered with in transit. On a domain registered through Cloudflare this is one click — Cloudflare publishes the DS record at the registrar for you. If the domain is registered elsewhere and only uses Cloudflare for DNS, copy the DS record Cloudflare shows you into your registrar's control panel by hand, or DNSSEC stays inactive.

#### eBay

1. **Use an email address with a custom domain.** eBay's developer account signup autoblocks free email providers.
2. **Register at [developer.ebay.com](https://developer.ebay.com)** using `developer@yournewdomain.com`. Approval normally arrives by confirmation email within 48 hours.
3. **Use a sandbox keyset.** while testing the app
4. **Mint a token** to check the credentials:
   ```bash
   curl -X POST https://api.sandbox.ebay.com/identity/v1/oauth2/token \
     -H "Authorization: Basic $(printf '%s:%s' "$CLIENT_ID" "$CLIENT_SECRET" | base64 -w0)" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d 'grant_type=client_credentials&scope=https://api.ebay.com/oauth/api_scope'
   ```
   The scope string is `https://api.ebay.com/oauth/api_scope` on both sandbox and production. Using `buy.browse` returns `invalid_scope`.
5. **Create a production keyset** when you are ready to run against real inventory. This one requires completing eBay's marketplace account-deletion notification compliance — a public HTTPS endpoint that echoes eBay's challenge code — and the keyset stays disabled until that passes. The domain from step 1 covers the endpoint.

> Sandbox inventory is synthetic: it has no real GPU listings and no meaningful prices. It verifies auth and response shape only. Real price data comes from Reddit during development, and from eBay once the production keyset is live.

#### Reddit

1. Use a Reddit account with a **verified email address**.
2. Go to [reddit.com/prefs/apps](https://www.reddit.com/prefs/apps) → **create another app**.
3. Choose type **script**. Set the redirect uri to `http://localhost:8080` — a script app never uses it, but the field is required.
4. Record the **client id** (the string shown under the app name) and the **secret**.
5. Set a descriptive User-Agent such as `gpuwatch/0.1 (by /u/yourname)`. Reddit heavily throttles default or generic agents.
6. **Mint a token** to check the credentials:
   ```bash
   curl -X POST https://www.reddit.com/api/v1/access_token \
     -H "Authorization: Basic $(printf '%s:%s' "$CLIENT_ID" "$CLIENT_SECRET" | base64 -w0)" \
     -H "User-Agent: gpuwatch/0.1 (by /u/yourname)" \
     -d 'grant_type=client_credentials'
   ```

The free tier allows 100 requests per minute per client id, when `User-Agent` is specified in request header.

#### Telegram

<!-- TODO: BotFather, bot token, chat id -->

---

## Technical specifications

<!-- TODO: architecture -->

<!-- TODO: data model -->

### Listing normalization

eBay returns prices as numbers. r/hardwareswap does not — posts follow the `[H]`/`[W]` tag convention, but the asking price is usually written in prose in the post body rather than in the title.

Reddit posts are parsed in two stages:

1. **Regex** splits the `[H]`/`[W]` tags, drops trade-only posts, and looks for a price in the title (`$180`, `180 shipped`, `180 obo`).
2. **LLM fallback** if stage 1 finds no price, the post body goes to a cheap model, which returns the price as JSON.

The fallback is optional and off by default, so the bot and its test suite run with no API key. Each post is sent to the model at most once — the result is stored on the listing row, so the same post reappearing in `/new` on the next poll costs nothing. Extracted prices are validated against the search term's current median before they can trigger an alert; anything outside the band is dropped and counted.

<!-- TODO: deal detection -->

<!-- TODO: deduplication -->

<!-- TODO: operations — logging, /healthz, backoff, graceful shutdown -->

<!-- TODO: known constraints -->

See [DESIGN_DOC.md](DESIGN_DOC.md) for the design architecture and [DEVELOPMENT_ACTION_PLAN.md](DEVELOPMENT_ACTION_PLAN.md) for the build sequence.
