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
  <a href="README.md"><img src="https://img.shields.io/badge/README-English-0078D4?style=flat-square" alt="English"></a>
</p>

<h1 align="center">cf-mail-api</h1>

<p align="center">
  基于 <b>Cloudflare Workers + D1</b> 的 webhook 临时邮箱后端。<br>
  <b>零 DNS、零路由、零真实域名。</b>HTTP 推入，HTTP 读出。
</p>

---

> **关键词：** Cloudflare Workers、Cloudflare D1、webhook 邮箱、可丢弃邮箱、临时邮箱、自建邮件后端、API 优先、零配置邮件存储、MailHog 替代、dev mail 捕获

---

## 这是什么

`cf-mail-api` 是一个 **无状态的 HTTP 邮件捕获后端**，完全跑在 Cloudflare 免费额度上。

核心路径就是一次 HTTP 调用：

```
POST /api/inbound   →   写入 D1   →   GET 读回
```

完事。**不需要** DNS 记录、不需要 MX 记录、不需要 Email Routing、不需要真实域名所有权、不需要邮件服务器。所谓「邮箱地址」(`xxx@mail.<你的域名>`) 只是 D1 里的一行字符串——域名那部分根本不需要真能收信。

典型用途：

| 场景 | 怎么用 |
|---|---|
| **注册一次性邮箱** | 生成 mailbox → 去某站注册 → 轮询 `GET /api/emails?email=...` 拿验证邮件 |
| **本地开发邮件捕获** | 把测试框架的 `mail()` 调用（或者任意 HTTP 发件方）推到 `POST /api/inbound` |
| **MailHog / Mailpit 替代** | 同样 UX，但跑在 CF 托管基础设施上——不用本地容器 |
| **API-first 邮件存储** | 任何上游服务用 webhook 就能往里存邮件 |
| **个人临时邮箱后端** | 生成 → 用 → 丢弃 |

如果你想叠加真 SMTP / Email Routing / 转发，参见 [可选扩展](#可选扩展)。但 webhook 模式才是主打卖点，不是降级方案。

---

## 部署前必读

> **不要**把任何人的 worker URL 拿来直接用。**不要**把部署好的 URL 发到任何公开渠道。**不要**用默认的 `*.workers.dev` 当公共邮箱服务。
>
> 本项目运行在 Cloudflare **免费额度**（约 10 万请求/天）。一旦别人找到你的端点，你的额度会被瞬间刷光，**你自己的邮箱就直接挂了**。
>
> 本项目作者 **不提供** 任何官方在线 demo。如果你在网上看到一个号称「官方 cf-mail-api」的站点，那不是我们——那是钓鱼/滥用镜像。永远自己部署。

---

## 仓库内容

| 组件 | 用途 |
|---|---|
| `src/index.js` | 主 worker——`POST /api/inbound` 写入器 + 邮箱查询 API |
| `src/send.js` | 可选的发件路由（Resend） |
| `schema.sql` | D1 数据库结构（mailboxes / messages） |
| `wrangler.toml` | Worker 配置——**部署前必须替换占位符** |
| `cloudflare_mail_client.py` | 可选 Python 客户端 |
| `LICENSE` | GNU GPL-3.0 |
| `README.md` | English documentation |
| `README.zh.md` | 中文文档（本文件） |

---

## 技术栈

| 层 | 选型 |
|---|---|
| 运行时 | Cloudflare Workers（V8 isolates）|
| 存储 | Cloudflare D1（SQLite）|
| 入站 | **HTTP webhook**（`POST /api/inbound`）——零 DNS / 零 MX |
| 入站（可选）| Cloudflare Email Routing → Worker |
| 发件（可选）| Resend HTTP API |
| 鉴权 | Bearer token / `x-api-key` / `?api_key=*** |
| 客户端（可选）| Python 3（`requests`）|

无 Node 依赖、无框架、无构建步骤。纯 `wrangler deploy`。

---

## 快速开始

### 1. 准备

- 一个 Cloudflare 账号（免费版即可）
- `wrangler` CLI：`npm i -g wrangler`
- Node.js 18+
- **不需要域名。** 个人用直接用默认的 `*.workers.dev` 路由就行。

### 2. 克隆 & 安装

```bash
git clone https://github.com/lingion/cf-mail-api.git
cd cf-mail-api
npm install
```

### 3. 创建 D1 数据库

```bash
wrangler d1 create mail_api
# 把打印出来的 database_id 填到 wrangler.toml
wrangler d1 execute mail_api --remote --file=./schema.sql
```

### 4. 配置 `wrangler.toml`

唯一必填项是 `API_TOKEN`。`[vars]` 里其他全是可选，参见 [可选扩展](#可选扩展)。

```toml
# 最小配置——替换 <your-d1-database-id> 和 <your-api-token>：
name = "cf-mail-api"
main = "src/index.js"
compatibility_date = "2026-03-22"

[[d1_databases]]
binding = "DB"
database_name = "mail_api"
database_id = "<your-d1-database-id>"

[vars]
API_TOKEN = "<用 openssl rand -hex 32 生成>"
```

如果想绑定自定义域名（只有 [扩展 A：Email Routing](#扩展-a--真-smtp--email-routing) 需要）：

```toml
[[routes]]
pattern = "api.<your-domain>/*"
zone_name = "<你的域名>"
```

### 5. 部署

```bash
wrangler deploy
```

核心配置到此结束，可以直接调 API 了。

### 6. 烟雾测试

```bash
# 1. 健康检查
curl https://<your-worker>.<your-subdomain>.workers.dev/health

# 2. 生成一个 mailbox
curl -X POST https://<your-worker>.<your-subdomain>.workers.dev/api/generate-email \
  -H 'x-api-key: ***' \
  -H 'Content-Type: application/json' \
  -d '{"prefix":"task_demo01","label":"注册测试","ttl_hours":24}'

# 3. 通过 webhook 推一条邮件进去
curl -X POST https://<your-worker>.<your-subdomain>.workers.dev/api/inbound \
  -H 'x-api-key: ***' \
  -H 'Content-Type: application/json' \
  -d '{
    "from":    "noreply@example.org",
    "to":      "task_demo01@mail.<your-domain>",
    "subject": "验证你的账号",
    "text":    "点击链接验证..."
  }'

# 4. 读回
curl 'https://<your-worker>.<your-subdomain>.workers.dev/api/emails?email=task_demo01@mail.<your-domain>' \
  -H 'x-api-key: ***'
```

你现在有了一个完整可用的临时邮箱后端。没有 DNS、没有路由、没有真实邮件服务器参与。

---

## API 参考

> 所有接口需要鉴权，三选一：
> `Authorization: Bearer <API_TOKEN>` · `x-api-key: ***` · `?api_key=<API_TOKEN>`

| 方法 | 路径 | 用途 |
|---|---|---|
| GET | `/health` | 健康检查 |
| POST | `/api/generate-email` | 创建一个新 mailbox |
| GET | `/api/mailboxes` | 列出所有 mailbox |
| GET | `/api/mailboxes/:id/messages` | 列出 mailbox 下的邮件 |
| GET | `/api/mailboxes/:id/messages/:msg_id` | 读取单封邮件 |
| GET | `/api/emails?email=...` | 按地址列出邮件 |
| GET | `/api/email/:id` | 按 id 读取单封 |
| DELETE | `/api/email/:id` | 删除单封 |
| DELETE | `/api/emails/clear?email=...` | 清空某地址下所有邮件 |
| GET | `/api/stats` | 计数（mailbox / message） |
| **POST** | **`/api/inbound`** | **Webhook——把邮件存入 D1** |

### POST /api/inbound（核心路径）

核心场景下你只需要这一个接口。任何能做 HTTP POST 的系统都能往里存邮件。

```bash
curl -X POST https://<your-worker>.<your-subdomain>.workers.dev/api/inbound \
  -H 'x-api-key: ***' \
  -H 'Content-Type: application/json' \
  -d '{
    "from":    "alice@example.org",
    "to":      "task_demo01@mail.<your-domain>",
    "subject": "验证你的账号",
    "text":    "点击链接验证...",
    "html":    "<a href=\"...\">验证</a>"
  }'
```

接受字段（除 `to` 之外全可选）：

| 字段 | 别名 | 说明 |
|---|---|---|
| `to` | `to_addr`, `recipient` | 目标 mailbox 地址（字符串或数组——取首个元素）|
| `from` | `from_addr` | 发件人（字符串或数组——取首个元素）|
| `subject` | — | 主题 |
| `text` | `text_body`, `body` | 纯文本正文 |
| `html` | `html_body` | HTML 正文 |
| `id` | `external_id` | 可选的外部消息 id，用于去重 / 关联 |

如果 `to` 指向的 mailbox 在 D1 里还没记录，worker 会自动创建。所以你可以往**任意**地址推邮件，不需要先调 `generate-email`。

### 生成 mailbox（可选）

```bash
curl -X POST https://<your-worker>.<your-subdomain>.workers.dev/api/generate-email \
  -H 'x-api-key: ***' \
  -H 'Content-Type: application/json' \
  -d '{"prefix":"task_demo01","label":"注册测试","ttl_hours":24}'
```

| 字段 | 规则 |
|---|---|
| `prefix` / `name` | 可选，若提供须匹配 `^[a-z0-9_-]{6,40}$` |
| `label` | 自由标签 |
| `ttl_hours` | mailbox 有效期，默认 24 小时 |

这只是个拿「带元数据的 mailbox 记录」的便利工具。webhook **接受任意地址**，不要求先注册。

### 读回邮件

```bash
# 某地址下的所有邮件
curl 'https://<your-worker>.<your-subdomain>.workers.dev/api/emails?email=task_demo01@mail.<your-domain>' \
  -H 'x-api-key: ***'

# 单封
curl 'https://<your-worker>.<your-subdomain>.workers.dev/api/email/<message_id>' \
  -H 'x-api-key: ***'
```

### 删除 / 清空

```bash
curl -X DELETE 'https://<your-worker>.<your-subdomain>.workers.dev/api/email/<message_id>' \
  -H 'x-api-key: ***'

curl -X DELETE 'https://<your-worker>.<your-subdomain>.workers.dev/api/emails/clear?email=<addr>@mail.<your-domain>' \
  -H 'x-api-key: ***'
```

### 统计

```bash
curl 'https://<your-worker>.<your-subdomain>.workers.dev/api/stats' -H 'x-api-key: ***'
```

---

## 可选扩展

webhook 核心不需要这些，按需叠加。

### 扩展 A — 真 SMTP / Email Routing

绑定自定义域名 + 启用 Cloudflare Email Routing，让 Worker 也能接收真 SMTP 服务器投到 `xxx@mail.<你的域名>` 的邮件。

1. 把域名加到 Cloudflare（DNS 必须在 CF）。
2. 在 `wrangler.toml` 加路由：
   ```toml
   [[routes]]
   pattern = "api.<你的域名>/*"
   zone_name = "<你的域名>"
   ```
3. 设置 `[vars] MAIL_DOMAIN = "mail.<你的域名>"`。
4. Cloudflare 控制台 → 你的域名 → **Email → Email Routing → Enable**，加 catch-all 路由 `*@mail.<你的域名>` → **Send to Worker** → `cf-mail-api`。

真实 SMTP 投来的邮件会被 Cloudflare 派给 worker，走同一条路径写 D1，查询接口不变。

### 扩展 B — 转发到真实邮箱

邮件进 D1 之后，可选地再转发一份到你的真实邮箱（比如 QQ / Gmail），让你不用轮询 API 就能看到。

在 `wrangler.toml`：

```toml
[vars]
FORWARD_TO_EMAIL = "you@gmail.com"
```

Worker 会把每条入站消息 POST 给一个小转发器（你可以接 SMTP 中继、Mailgun、Resend、或别的）。`FORWARD_TO_EMAIL` 只是个目标字符串，worker 读取它。

### 扩展 C — Resend 发件

加第三个路由 `send.<你的域名>`，设置 Resend API key：

```toml
[vars]
RESEND_API_KEY = "***"
```

```bash
curl -X POST https://send.<你的域名>/api/send \
  -H 'x-api-key: ***' \
  -H 'Content-Type: application/json' \
  -d '{
    "from":    "task_demo01@mail.<你的域名>",
    "to":      "bob@example.org",
    "subject": "你好",
    "text":    "通过 cf-mail-api 发出"
  }'
```

`from` 必须是 D1 里已存在的 mailbox，且域名要有合法的 SPF/DKIM 记录才能保证送达率。

---

## 配置速查

| 环境变量 | 是否必填 | 用途 |
|---|---|---|
| `API_TOKEN` | **是** | API Bearer token。用 `openssl rand -hex 32` 生成。 |
| `MAIL_DOMAIN` | 否 | 生成 mailbox 地址时显示的域名（如 `mail.<你的域名>`）。纯展示——邮件路由仍然走 webhook。 |
| `FORWARD_TO_EMAIL` | 否 | 入站邮件副本的转发目标（扩展 B）。 |
| `RESEND_API_KEY` | 否 | 发件服务商 key（扩展 C）。 |

---

## 成本 & 配额

本项目设计为完全跑在 Cloudflare **免费额度**内：

| 资源 | 免费额度 |
|---|---|
| Workers 请求 | 100,000 / 天 |
| D1 读 | 5,000,000 / 天 |
| D1 写 | 100,000 / 天 |
| Email Routing 邮件 | 100 / 天（每个目标地址，仅扩展 A）|

**切勿**把此服务对外公开。每个外部请求都在消耗你的配额。如果你需要更大空间，请在前面加一层鉴权（每个用户独立 token、IP 白名单、或速率限制）——鉴权开关项目里已经预留好，别分享 token 就行。

---

## 仓库规则

`lingion/cf-mail-api` 是本项目的 **唯一主线**。任何协作副本或 fork（即便用于测试）都不应替代本仓作为主入口。项目所有有意义的演进都最终回到这里。

---

## 文档

- `README.md` — English documentation
- `README.zh.md` — 中文文档（本文件）
- `RESEND_SETUP.md` — 历史 Resend / 发件配置笔记
- `schema.sql` — D1 数据库结构参考

---

## 许可证

GNU 通用公共许可证 v3.0。详见 [LICENSE](./LICENSE)。

简言之：你可以自由使用、修改、再分发——包括商业用途——但**任何衍生作品也必须 GPL-3.0** 且**必须保留版权声明**。无任何担保。

---

## 贡献

欢迎在 <https://github.com/lingion/cf-mail-api> 提 PR。提交贡献即视为同意同样以 GPL-3.0 授权。