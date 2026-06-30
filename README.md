<p align="center">
  <a href="https://github.com/lingion/cf-mail-api/stargazers"><img src="https://img.shields.io/github/stars/lingion/cf-mail-api?style=for-the-badge&logo=github&color=FFD700" alt="Stars"></a>
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
  Self-hosted temporary mailbox service on <b>Cloudflare Workers + D1</b> with your own custom subdomain.<br>
  Generate any <code>xxx@&lt;your-domain&gt;</code> on the fly, receive mail into D1, forward to your real inbox.
</p>

---

> **Keywords:** Cloudflare Workers, Cloudflare D1, Email Routing, disposable email, temporary mailbox, custom subdomain, self-hosted mail server, free tier, forward mail, Resend API

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
| `src/index.js` | Main worker — inbound mail handler + mailbox API |
| `src/send.js` | Optional outbound send route via Resend |
| `schema.sql` | D1 database schema (mailboxes, messages) |
| `wrangler.toml` | Worker config — **replace placeholders before deploy** |
| `cloudflare_mail_client.py` | Optional Python client for the API |
| `LICENSE` | GNU GPL-3.0 |
| `README.md` | English (this file) |
| `README.zh.md` | 中文文档 |

---

## Tech Stack

| Layer | Choice |
|---|---|
| Runtime | Cloudflare Workers (V8 isolates) |
| Storage | Cloudflare D1 (SQLite) |
| Inbound | Cloudflare Email Routing → Worker |
| Outbound (optional) | Resend HTTP API |
| Auth | Bearer token / `x-api-key` / `?api_key=` |
| Client (optional) | Python 3 (`requests`) |

No Node dependencies. No framework. No build step. Pure `wrangler deploy`.

---

## Quick Start

### 1. Prerequisites

- A domain managed by Cloudflare (DNS must be on CF — needed for Email Routing)
- A Cloudflare account (free tier is enough)
- `wrangler` CLI: `npm i -g wrangler`
- Node.js 18+

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

Replace **every** `<your-domain>` placeholder with your own CF-managed domain. Replace `<your-d1-database-id>`, `<your-api-token>`, `<your-real-inbox@example.com>` with your own values.

The **mailbox receiving domain** (`mail.<your-domain>`), the **API domain** (`api.<your-domain>`), and the **send domain** (`send.<your-domain>`) should all live under one zone you control.

```toml
# Example — DO NOT copy these values literally:
[[routes]]
pattern = "api.example.com/*"
zone_name = "example.com"

[vars]
MAIL_DOMAIN = "mail.example.com"
API_TOKEN = "<generate-with: openssl rand -hex 32>"
FORWARD_TO_EMAIL = "you@gmail.com"
```

### 5. Enable Cloudflare Email Routing

In the Cloudflare dashboard for your zone:

1. **Email → Email Routing → Enable**
2. Add a destination address (your real inbox) and verify it
3. Add a catch-all route: `*@mail.<your-domain>` → **Send to Worker** → select `cf-mail-api`

The Worker handles the rest — writes to D1, optionally forwards.

### 6. Deploy

```bash
wrangler deploy
```

After deploy, your private API lives at `https://api.<your-domain>/` — **don't share it**.

---

## API Reference

> All endpoints require auth via one of:
> `Authorization: Bearer <API_TOKEN>` · `x-api-key: ***` · `?api_key=<API_TOKEN>`

### Health check

```bash
curl https://api.<your-domain>/health
```

### Generate a mailbox

```bash
curl -X POST https://api.<your-domain>/api/generate-email \
  -H 'x-api-key: ***' \
  -H 'Content-Type: application/json' \
  -d '{"prefix":"task_20260630_ab12","label":"signup","ttl_hours":24}'
```

Optional fields:

| field | rule |
|---|---|
| `prefix` / `name` | optional, must match `^[a-z0-9_-]{6,40}$` if provided |
| `label` | free-form tag |
| `ttl_hours` | mailbox lifetime, default 24h |

Response:

```json
{
  "success": true,
  "data": {
    "mailbox_id": "task_20260630_ab12",
    "email": "task_20260630_ab12@mail.<your-domain>",
    "domain": "mail.<your-domain>",
    "token": "***",
    "created_at": "...",
    "expires_at": "..."
  }
}
```

### List messages for a mailbox

```bash
curl 'https://api.<your-domain>/api/mailboxes/<mailbox_id>/messages' \
  -H 'x-api-key: ***'
```

### Fetch / delete a single message

```bash
curl 'https://api.<your-domain>/api/email/<message_id>'  -H 'x-api-key: ***'
curl -X DELETE 'https://api.<your-domain>/api/email/<message_id>' -H 'x-api-key: ***'
```

### Clear all mail for an address

```bash
curl -X DELETE 'https://api.<your-domain>/api/emails/clear?email=<addr>@mail.<your-domain>' \
  -H 'x-api-key: ***'
```

### Stats

```bash
curl 'https://api.<your-domain>/api/stats' -H 'x-api-key: ***'
```

### Inbound webhook (for external senders)

```bash
curl -X POST 'https://api.<your-domain>/api/inbound' \
  -H 'x-api-key: ***' \
  -H 'Content-Type: application/json' \
  -d '{
    "from": "alice@example.org",
    "to": "task_20260630_ab12@mail.<your-domain>",
    "subject": "Verify your account",
    "text": "Click here to verify..."
  }'
```

Supported inbound fields: `id` / `external_id`, `from` / `from_addr`, `to` / `to_addr` / `recipient`, `subject`, `text` / `text_body`, `html` / `html_body`.

---

## Sending Email (optional)

If you want to **send** mail from a temp address too, configure a third route `send.<your-domain>` and set a Resend API key in `wrangler.toml`:

```toml
[vars]
RESEND_API_KEY = "***"
```

Then:

```bash
curl -X POST https://send.<your-domain>/api/send \
  -H 'x-api-key: ***' \
  -H 'Content-Type: application/json' \
  -d '{
    "from":    "task_20260630_ab12@mail.<your-domain>",
    "to":      "bob@example.org",
    "subject": "Hello from a temp mailbox",
    "text":    "This message was sent via cf-mail-api."
  }'
```

**Note:** `from` must be a mailbox that already exists in your D1, and the domain must have a valid SPF/DKIM record for deliverability.

---

## Configuration Cheatsheet

| Env var | Purpose |
|---|---|
| `MAIL_DOMAIN` | The `<subdomain>.<your-domain>` that receives mail |
| `API_TOKEN` | Bearer token for the public API (rotate periodically) |
| `FORWARD_TO_EMAIL` | (optional) Where to forward inbound mail |
| `RESEND_API_KEY` | (optional) Outbound provider; only needed if you enable send |

---

## Cost & Quota

This project is designed to run entirely on Cloudflare's **free tier**:

| Resource | Free tier |
|---|---|
| Workers requests | 100,000 / day |
| D1 reads | 5,000,000 / day |
| D1 writes | 100,000 / day |
| Email Routing messages | 100 / day (per destination) |

**Do not** expose this service publicly. Every external request consumes your quota. If you need more headroom, put an auth layer in front (a per-user token, IP allowlist, or rate limit) — the auth flag is already there, just don't share the token.

---

## Repository Rule

`lingion/cf-mail-api` is the **only mainline** source of truth for this project. Any collaboration mirror or fork (e.g. for testing) should not replace this repo as the primary landing page. All meaningful project evolution lands here.

---

## Docs

- `README.md` — English (this file)
- `README.zh.md` — 中文文档
- `RESEND_SETUP.md` — Resend / outbound setup notes
- `schema.sql` — D1 database schema reference

---

## License

GNU General Public License v3.0. See [LICENSE](./LICENSE).

In short: you can use, modify, and redistribute this freely — including commercially — but **any derivative work must also be GPL-3.0** and **must keep the copyright notice**. There is no warranty.

---

## Contributing

PRs welcome at <https://github.com/lingion/cf-mail-api>. By contributing you agree your contribution is also licensed under GPL-3.0.