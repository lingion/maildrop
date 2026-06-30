<p align="center">
  <a href="https://img.shields.io/github/stars/lingion/cf-mail-api?style=for-the-badge&logo=github&color=FFD700"><img src="https://img.shields.io/github/stars/lingion/cf-mail-api?style=for-the-badge&logo=github&color=FFD700" alt="Stars"></a>
  <a href="https://github.com/lingion/cf-mail-api/network/members"><img src="https://img.shields.io/github/forks/lingion/cf-mail-api?style=for-the-badge&logo=github&color=8B5CF6" alt="Forks"></a>
  <a href="https://github.com/lingion/cf-mail-api/issues"><img src="https://img.shields.io/github/issues/lingion/cf-mail-api?style=for-the-badge&logo=github&color=EF4444" alt="Issues"></a>
  <a href="https://github.com/lingion/cf-mail-api/blob/main/LICENSE"><img src="https://img.shields.io/github/license/lingion/cf-mail-api?style=for-the-badge&logo=github&color=10B981" alt="License"></a>
  <br>
  <a href="https://github.com/lingion/cf-mail-api/commits/main"><img src="https://img.shields.io/github/last-commit/lingion/cf-mail-api?style=flat-square" alt="Last commit"></a>
  <img src="https://img.shields.io/badge/runtime-Cloudflare%20Workers-F38020?style=flat-square&logo=cloudflare&logoColor=white" alt="CF Workers">
  <img src="https://img.shields.io/badge/storage-Cloudflare%20D1-FF7043?style=flat-square&logo=cloudflare&logoColor=white" alt="D1">
  <img src="https://img.shields.io/badge/lang-JavaScript-F7DF1E?style=flat-square&logo=javascript&logoColor=black" alt="JS">
  <a href="README.zh.md"><img src="https://img.shields.io/badge/README-中文-CC0000?style=flat-square" alt="中文"></a>
</p>

<h1 align="center">cf-mail-api</h1>

<p align="center">
  A webhook-based disposable mailbox backend on <b>Cloudflare Workers + D1</b>.<br>
  <b>Zero DNS, zero routing, zero real domain required.</b> Push mail in via HTTP, read it back via HTTP.
</p>

---

> **Keywords:** Cloudflare Workers, Cloudflare D1, webhook mailbox, disposable email, temporary mailbox, self-hosted mail backend, API-first, zero-config mail storage, MailHog alternative, dev mail capture

---

## What This Is

`cf-mail-api` is a **stateless HTTP mail-capture backend** that lives entirely on Cloudflare's free tier.

The core path is one HTTP call:

```
POST /api/inbound   →   write to D1   →   fetch back via GET
```

That's it. There is **no required** DNS record, MX record, Email Routing setup, real domain ownership, or mail server. The "mailbox address" (`xxx@mail.<your-domain>`) is just a string identifier in D1 — the domain part does not need to actually receive mail.

This makes it useful as:

| Use case | How |
|---|---|
| **Disposable inbox for signups** | Generate mailbox → register somewhere → poll `GET /api/emails?email=...` for the verification mail |
| **Local dev mail capture** | Pipe your test framework's `mail()` calls (or any HTTP sender) into `POST /api/inbound` |
| **MailHog / Mailpit replacement** | Same UX, but on managed CF infra — no local container to run |
| **API-first mail store** | Any upstream service with an HTTP webhook can deposit mail here |
| **Personal temp-mail backend** | Generate → use → discard |

If you want real-world SMTP / Email Routing / forwarding on top of the same backend, see [Optional Add-ons](#optional-add-ons). But the webhook-only mode is the headline feature, not a fallback.

---

## Before You Deploy — Read This First

> **DO NOT** point this at anyone else's worker URL. **DO NOT** publish your deployed URL publicly. **DO NOT** use the default `*.workers.dev` URL as a shared service.
>
> This project runs on Cloudflare's **free tier** (~100k requests/day). The moment someone else finds your endpoint, they will burn through your quota and **your own mailbox stops working**.
>
> The author of this project **does not** publish a hosted demo. If you find one online claiming to be "the official cf-mail-api", it is **not** us — it is a phishing/abuse mirror. Always deploy your own.

---

## What Is in This Repository

| Component | Purpose |
|---|---|
| `src/index.js` | Main worker — `POST /api/inbound` writer + mailbox query API |
| `src/send.js` | Optional outbound send route via Resend |
| `schema.sql` | D1 schema (mailboxes, messages) |
| `wrangler.toml` | Worker config — **replace placeholders before deploy** |
| `cloudflare_mail_client.py` | Optional Python client |
| `LICENSE` | GNU GPL-3.0 |
| `README.md` | English (this file) |
| `README.zh.md` | 中文文档 |

---

## Tech Stack

| Layer | Choice |
|---|---|
| Runtime | Cloudflare Workers (V8 isolates) |
| Storage | Cloudflare D1 (SQLite) |
| Inbound | **HTTP webhook** (`POST /api/inbound`) — no DNS / MX required |
| Inbound (optional) | Cloudflare Email Routing → Worker |
| Outbound (optional) | Resend HTTP API |
| Auth | Bearer token / `x-api-key` / `?api_key=*** |
| Client (optional) | Python 3 (`requests`) |

No Node dependencies. No framework. No build step. Pure `wrangler deploy`.

---

## Quick Start

### 1. Prerequisites

- A Cloudflare account (free tier is enough)
- `wrangler` CLI: `npm i -g wrangler`
- Node.js 18+
- **No domain required.** You can run the whole thing on the default `*.workers.dev` route for personal use.

### 2. Clone & install

```bash
git clone https://github.com/lingion/cf-mail-api.git
cd cf-mail-api
npm install
```

### 3. Create the D1 database

```bash
wrangler d1 create mail_api
# Copy the printed `database_id` into wrangler.toml
wrangler d1 execute mail_api --remote --file=./schema.sql
```

### 4. Configure `wrangler.toml`

The only mandatory thing is your `API_TOKEN`. Everything else in `[vars]` is optional — see [Optional Add-ons](#optional-add-ons).

```toml
# Minimal config — replace <your-d1-database-id> and <your-api-token>:
name = "cf-mail-api"
main = "src/index.js"
compatibility_date = "2026-03-22"

[[d1_databases]]
binding = "DB"
database_name = "mail_api"
database_id = "<your-d1-database-id>"

[vars]
API_TOKEN = "<generate-with: openssl rand -hex 32>"
```

To bind a custom domain (optional — only needed for the [Email Routing add-on](#optional-add-ons)):

```toml
[[routes]]
pattern = "api.<your-domain>/*"
zone_name = "<your-domain>"
```

### 5. Deploy

```bash
wrangler deploy
```

That's the entire core setup. You're done — start hitting the API.

### 6. Smoke test

```bash
# 1. Health check
curl https://<your-worker>.<your-subdomain>.workers.dev/health

# 2. Generate a mailbox
curl -X POST https://<your-worker>.<your-subdomain>.workers.dev/api/generate-email \
  -H 'x-api-key: ***' \
  -H 'Content-Type: application/json' \
  -d '{"prefix":"task_demo01","label":"signup-test","ttl_hours":24}'

# 3. Push a message in via the webhook
curl -X POST https://<your-worker>.<your-subdomain>.workers.dev/api/inbound \
  -H 'x-api-key: ***' \
  -H 'Content-Type: application/json' \
  -d '{
    "from":    "noreply@example.org",
    "to":      "task_demo01@mail.<your-domain>",
    "subject": "Verify your account",
    "text":    "Click here to verify..."
  }'

# 4. Read it back
curl 'https://<your-worker>.<your-subdomain>.workers.dev/api/emails?email=task_demo01@mail.<your-domain>' \
  -H 'x-api-key: ***'
```

You now have a fully working disposable mailbox backend. No DNS, no routing, no real mail server involved.

---

## API Reference

> All endpoints require auth via one of:
> `Authorization: Bearer <API_TOKEN>` · `x-api-key: ***` · `?api_key=<API_TOKEN>`

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | Health check |
| POST | `/api/generate-email` | Create a new mailbox |
| GET | `/api/mailboxes` | List all mailboxes |
| GET | `/api/mailboxes/:id/messages` | List messages in a mailbox |
| GET | `/api/mailboxes/:id/messages/:msg_id` | Fetch one message |
| GET | `/api/emails?email=...` | List messages for an address |
| GET | `/api/email/:id` | Fetch one message by id |
| DELETE | `/api/email/:id` | Delete one message |
| DELETE | `/api/emails/clear?email=...` | Clear all mail for an address |
| GET | `/api/stats` | Counts (mailboxes, messages) |
| **POST** | **`/api/inbound`** | **Webhook — deposit a message into D1** |

### POST /api/inbound (the core path)

This is the only endpoint you actually need for the core use case. Any system that can do an HTTP POST can deposit mail here.

```bash
curl -X POST https://<your-worker>.<your-subdomain>.workers.dev/api/inbound \
  -H 'x-api-key: ***' \
  -H 'Content-Type: application/json' \
  -d '{
    "from":    "alice@example.org",
    "to":      "task_demo01@mail.<your-domain>",
    "subject": "Verify your account",
    "text":    "Click here to verify...",
    "html":    "<a href=\"...\">Verify</a>"
  }'
```

Accepted fields (all optional except `to`):

| Field | Aliases | Notes |
|---|---|---|
| `to` | `to_addr`, `recipient` | Target mailbox address (string or array — first element used) |
| `from` | `from_addr` | Sender (string or array — first element used) |
| `subject` | — | Mail subject |
| `text` | `text_body`, `body` | Plain-text body |
| `html` | `html_body` | HTML body |
| `id` | `external_id` | Optional external message id for de-dup / correlation |

If `to` is a mailbox that does not yet exist in D1, the worker auto-creates it. So you can deposit mail into any arbitrary address without calling `generate-email` first.

### Generate a mailbox (optional)

```bash
curl -X POST https://<your-worker>.<your-subdomain>.workers.dev/api/generate-email \
  -H 'x-api-key: ***' \
  -H 'Content-Type: application/json' \
  -d '{"prefix":"task_demo01","label":"signup-test","ttl_hours":24}'
```

| field | rule |
|---|---|
| `prefix` / `name` | optional, must match `^[a-z0-9_-]{6,40}$` if provided |
| `label` | free-form tag |
| `ttl_hours` | mailbox lifetime, default 24h |

This is purely a convenience helper for getting a tracked mailbox record. The webhook accepts mail into **any** address, registered or not.

### Read mail back

```bash
# All messages for an address
curl 'https://<your-worker>.<your-subdomain>.workers.dev/api/emails?email=task_demo01@mail.<your-domain>' \
  -H 'x-api-key: ***'

# Single message
curl 'https://<your-worker>.<your-subdomain>.workers.dev/api/email/<message_id>' \
  -H 'x-api-key: ***'
```

### Delete / clear

```bash
curl -X DELETE 'https://<your-worker>.<your-subdomain>.workers.dev/api/email/<message_id>' \
  -H 'x-api-key: ***'

curl -X DELETE 'https://<your-worker>.<your-subdomain>.workers.dev/api/emails/clear?email=<addr>@mail.<your-domain>' \
  -H 'x-api-key: ***'
```

### Stats

```bash
curl 'https://<your-worker>.<your-subdomain>.workers.dev/api/stats' -H 'x-api-key: ***'
```

---

## Optional Add-ons

The webhook core works without any of these. Add them only if you want them.

### Add-on A — Real SMTP / Email Routing

Bind a custom domain and enable Cloudflare Email Routing so the Worker also accepts mail that real SMTP servers deliver to `xxx@mail.<your-domain>`.

1. Add a domain to Cloudflare (DNS must be on CF).
2. In `wrangler.toml`, add a route:
   ```toml
   [[routes]]
   pattern = "api.<your-domain>/*"
   zone_name = "<your-domain>"
   ```
3. Set `[vars] MAIL_DOMAIN = "mail.<your-domain>"`.
4. In Cloudflare dashboard → your zone → **Email → Email Routing → Enable**, then add a catch-all route `*@mail.<your-domain>` → **Send to Worker** → `cf-mail-api`.

Mail delivered by real SMTP is dispatched to the worker by Cloudflare, written to D1 via the same path, and becomes queryable via the same API.

### Add-on B — Forward to a real inbox

After mail lands in D1, optionally forward a copy to your real address (e.g. QQ / Gmail) so you see it without polling the API.

In `wrangler.toml`:

```toml
[vars]
FORWARD_TO_EMAIL = "you@gmail.com"
```

The Worker will POST every inbound message to a small forwarder (configure with whatever delivery you prefer — SMTP relay, Mailgun, Resend, etc.). `FORWARD_TO_EMAIL` itself is just the destination string; the worker reads it.

### Add-on C — Send outbound mail (Resend)

Add a third route `send.<your-domain>` and set a Resend API key:

```toml
[vars]
RESEND_API_KEY = "***"
```

```bash
curl -X POST https://send.<your-domain>/api/send \
  -H 'x-api-key: ***' \
  -H 'Content-Type: application/json' \
  -d '{
    "from":    "task_demo01@mail.<your-domain>",
    "to":      "bob@example.org",
    "subject": "Hello",
    "text":    "Sent via cf-mail-api"
  }'
```

`from` must be a mailbox that already exists in your D1, and the domain must have a valid SPF/DKIM record for deliverability.

---

## Configuration Cheatsheet

| Env var | Required? | Purpose |
|---|---|---|
| `API_TOKEN` | **yes** | Bearer token for the API. Generate with `openssl rand -hex 32`. |
| `MAIL_DOMAIN` | no | The display domain used when generating mailbox addresses (e.g. `mail.<your-domain>`). Cosmetic — mail routing still uses the webhook. |
| `FORWARD_TO_EMAIL` | no | Where to forward inbound mail copies (add-on B). |
| `RESEND_API_KEY` | no | Outbound provider key (add-on C). |

---

## Cost & Quota

This project is designed to run entirely on Cloudflare's **free tier**:

| Resource | Free tier |
|---|---|
| Workers requests | 100,000 / day |
| D1 reads | 5,000,000 / day |
| D1 writes | 100,000 / day |
| Email Routing messages | 100 / day (per destination, add-on A only) |

**Do not** expose this service publicly. Every external request consumes your quota. If you need more headroom, put an auth layer in front (a per-user token, IP allowlist, or rate limit) — the auth flag is already there, just don't share the token.

---

## Repository Rule

`lingion/cf-mail-api` is the **only mainline** source of truth for this project. Any collaboration mirror or fork (e.g. for testing) should not replace this repo as the primary landing page. All meaningful project evolution lands here.

---

## Docs

- `README.md` — English (this file)
- `README.zh.md` — 中文文档
- `RESEND_SETUP.md` — Legacy Resend / outbound setup notes
- `schema.sql` — D1 database schema reference

---

## License

GNU General Public License v3.0. See [LICENSE](./LICENSE).

In short: you can use, modify, and redistribute this freely — including commercially — but **any derivative work must also be GPL-3.0** and **must keep the copyright notice**. There is no warranty.

---

## Contributing

PRs welcome at <https://github.com/lingion/cf-mail-api>. By contributing you agree your contribution is also licensed under GPL-3.0.