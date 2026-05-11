# Deployment -- Telegram Bot

Patterns for deploying Telegram bots to production.
Polling = long-running process. Webhooks = HTTP endpoint behind reverse proxy or serverless.

---

## Table of Contents

1. VPS (systemd)
2. VPS (PM2)
3. Docker
4. Reverse Proxy (Webhooks)
5. Serverless
6. PaaS
7. Setting Webhook
8. Environment Variables

---

## 1. VPS (systemd)

Unit file -- `/etc/systemd/system/my-bot.service`:

```ini
[Unit]
Description=Telegram Bot (my-bot)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=botuser
Group=botuser
WorkingDirectory=/opt/my-bot
EnvironmentFile=/opt/my-bot/.env
ExecStart=/opt/my-bot/.venv/bin/python -m bot
Restart=on-failure
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=5

# Hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/my-bot/data
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

`.env` file (chmod 600, owned by botuser):

```bash
BOT_TOKEN=123456:ABC-DEF
DATABASE_URL=postgresql://bot:pass@localhost:5432/botdb
LOG_LEVEL=INFO
ADMIN_IDS=164795011
```

Commands:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now my-bot
sudo systemctl restart my-bot
journalctl -u my-bot -f
journalctl -u my-bot --since "1 hour ago"
```

Restart policy: `on-failure` + `RestartSec=5` prevents tight crash loops.
`StartLimitBurst=5` in 60s -- systemd stops restarting after 5 consecutive failures.

---

## 2. VPS (PM2)

### Python bot -- `ecosystem.config.js`

```js
module.exports = {
  apps: [
    {
      name: "my-bot",
      script: ".venv/bin/python",
      args: "-m bot",
      cwd: "/opt/my-bot",
      interpreter: "none",
      env: {
        BOT_TOKEN: "123456:ABC-DEF",
        LOG_LEVEL: "INFO",
      },
      max_restarts: 10,
      min_uptime: "10s",
      restart_delay: 5000,
      autorestart: true,
      log_date_format: "YYYY-MM-DD HH:mm:ss Z",
      error_file: "/opt/my-bot/logs/error.log",
      out_file: "/opt/my-bot/logs/out.log",
      merge_logs: true,
    },
  ],
};
```

### Node.js bot -- `ecosystem.config.js`

```js
module.exports = {
  apps: [
    {
      name: "my-bot",
      script: "dist/index.js",
      cwd: "/opt/my-bot",
      node_args: "--enable-source-maps",
      env: {
        NODE_ENV: "production",
        BOT_TOKEN: "123456:ABC-DEF",
        LOG_LEVEL: "info",
      },
      max_restarts: 10,
      min_uptime: "10s",
      restart_delay: 5000,
      autorestart: true,
      log_date_format: "YYYY-MM-DD HH:mm:ss Z",
    },
  ],
};
```

### PM2 commands

```bash
pm2 start ecosystem.config.js
pm2 restart my-bot
pm2 stop my-bot
pm2 logs my-bot --lines 100
pm2 save                       # persist process list across reboots
pm2 startup                    # generate systemd unit for PM2 itself
pm2 monit                      # real-time dashboard
```

---

## 3. Docker

### Dockerfile -- Python (aiogram)

```dockerfile
FROM python:3.12-slim AS base

WORKDIR /app

RUN groupadd -r bot && useradd -r -g bot bot

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY bot/ bot/

USER bot

CMD ["python", "-m", "bot"]
```

### Dockerfile -- Node.js (grammY)

```dockerfile
FROM node:22-slim AS builder

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --ignore-scripts
COPY tsconfig.json ./
COPY src/ src/
RUN npm run build

FROM node:22-slim

WORKDIR /app
RUN groupadd -r bot && useradd -r -g bot bot

COPY package.json package-lock.json ./
RUN npm ci --omit=dev --ignore-scripts
COPY --from=builder /app/dist dist/

USER bot

CMD ["node", "dist/index.js"]
```

### docker-compose.yml

```yaml
services:
  bot:
    build: .
    restart: unless-stopped
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - internal

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: bot
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      POSTGRES_DB: botdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    secrets:
      - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U bot -d botdb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - internal

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --appendonly yes --maxmemory 128mb --maxmemory-policy allkeys-lru
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - internal

volumes:
  pgdata:
  redisdata:

secrets:
  db_password:
    file: ./secrets/db_password.txt

networks:
  internal:
```

### .dockerignore

```
.git
.env
.venv
__pycache__
node_modules
*.pyc
*.pyo
.mypy_cache
.pytest_cache
dist
logs
*.log
.DS_Store
```

---

## 4. Reverse Proxy (Webhooks)

### Nginx

```nginx
server {
    listen 443 ssl http2;
    server_name bot.example.com;

    ssl_certificate     /etc/letsencrypt/live/bot.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/bot.example.com/privkey.pem;

    location /webhook/telegram {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Pass Telegram's secret token header for verification
        proxy_set_header X-Telegram-Bot-Api-Secret-Token $http_x_telegram_bot_api_secret_token;

        # Only allow Telegram IP ranges
        # https://core.telegram.org/bots/webhooks#the-short-version
        allow 149.154.160.0/20;
        allow 91.108.4.0/22;
        deny all;

        # Limit body size (Telegram updates are small)
        client_max_body_size 1m;
    }
}
```

### Caddy

```caddyfile
bot.example.com {
    route /webhook/telegram {
        reverse_proxy 127.0.0.1:8080 {
            header_up X-Telegram-Bot-Api-Secret-Token {header.X-Telegram-Bot-Api-Secret-Token}
        }
    }

    # Block everything else
    respond 404
}
```

Caddy handles TLS automatically via Let's Encrypt. No certificate config needed.

Secret token verification inside bot (Python example):

```python
WEBHOOK_SECRET = os.environ["WEBHOOK_SECRET"]

@app.post("/webhook/telegram")
async def webhook(request: Request):
    token = request.headers.get("X-Telegram-Bot-Api-Secret-Token")
    if not hmac.compare_digest(token or "", WEBHOOK_SECRET):
        raise HTTPException(status_code=403, detail="Invalid secret token")
    update = Update.model_validate(await request.json())
    await dp.feed_update(bot, update)
    return {"ok": True}
```

---

## 5. Serverless

### Cloudflare Workers (grammY)

`wrangler.toml`:

```toml
name = "my-bot"
main = "src/index.ts"
compatibility_date = "2024-01-01"
compatibility_flags = ["nodejs_compat"]

[vars]
LOG_LEVEL = "info"
```

`src/index.ts`:

```typescript
import { Bot, webhookCallback } from "grammy";

interface Env {
  BOT_TOKEN: string;
  WEBHOOK_SECRET: string;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const bot = new Bot(env.BOT_TOKEN);

    bot.command("start", (ctx) => ctx.reply("Hello from Cloudflare Workers"));

    bot.on("message:text", (ctx) => ctx.reply(`Echo: ${ctx.message.text}`));

    const handleUpdate = webhookCallback(bot, "cloudflare-mod", {
      secretToken: env.WEBHOOK_SECRET,
    });

    return handleUpdate(request);
  },
};
```

Secrets:

```bash
wrangler secret put BOT_TOKEN
wrangler secret put WEBHOOK_SECRET
wrangler deploy
```

### Supabase Edge Functions

Directory: `supabase/functions/telegram-bot/index.ts`

```typescript
import { serve } from "https://deno.land/std@0.177.0/http/server.ts";
import { Bot, webhookCallback } from "https://deno.land/x/grammy@v1.21.1/mod.ts";

const bot = new Bot(Deno.env.get("BOT_TOKEN")!);

bot.command("start", (ctx) => ctx.reply("Hello from Supabase Edge Functions"));

bot.on("message:text", (ctx) => ctx.reply(`Echo: ${ctx.message.text}`));

const handleUpdate = webhookCallback(bot, "std/http", {
  secretToken: Deno.env.get("WEBHOOK_SECRET"),
});

serve(async (req) => {
  if (req.method === "POST") {
    return handleUpdate(req);
  }
  return new Response("Method not allowed", { status: 405 });
});
```

Deploy:

```bash
supabase secrets set BOT_TOKEN=123456:ABC-DEF WEBHOOK_SECRET=mysecret
supabase functions deploy telegram-bot --no-verify-jwt
```

Webhook URL: `https://<project-ref>.supabase.co/functions/v1/telegram-bot`

### Yandex Cloud Functions (Python)

`handler.py`:

```python
import json
import os

from aiogram import Bot, Dispatcher, types

bot = Bot(token=os.environ["BOT_TOKEN"])
dp = Dispatcher()

@dp.message()
async def echo(message: types.Message) -> None:
    await message.reply(f"Echo: {message.text}")

async def handler(event: dict, context) -> dict:
    secret = event.get("headers", {}).get("X-Telegram-Bot-Api-Secret-Token", "")
    if secret != os.environ["WEBHOOK_SECRET"]:
        return {"statusCode": 403, "body": "Forbidden"}

    body = json.loads(event["body"])
    update = types.Update.model_validate(body)
    await dp.feed_update(bot, update)

    return {"statusCode": 200, "body": "ok"}
```

API Gateway `openapi.yaml`:

```yaml
openapi: 3.0.0
info:
  title: Telegram Bot Webhook
  version: 1.0.0
paths:
  /webhook:
    post:
      x-yc-apigateway-integration:
        type: cloud_functions
        function_id: d4e1234567890
        service_account_id: aje1234567890
      operationId: telegramWebhook
```

Deploy:

```bash
yc serverless function create --name my-bot
yc serverless function version create \
  --function-name my-bot \
  --runtime python312 \
  --entrypoint handler.handler \
  --memory 128m \
  --execution-timeout 10s \
  --source-path ./src \
  --environment BOT_TOKEN=123456:ABC-DEF,WEBHOOK_SECRET=mysecret
```

### Vercel

`api/webhook.ts`:

```typescript
import type { VercelRequest, VercelResponse } from "@vercel/node";
import { Bot, webhookCallback } from "grammy";
import crypto from "node:crypto";

const bot = new Bot(process.env.BOT_TOKEN!);

bot.command("start", (ctx) => ctx.reply("Hello from Vercel"));
bot.on("message:text", (ctx) => ctx.reply(`Echo: ${ctx.message.text}`));

const callback = webhookCallback(bot, "https");

export default async function handler(req: VercelRequest, res: VercelResponse) {
  if (req.method !== "POST") {
    return res.status(405).json({ error: "Method not allowed" });
  }

  const secret = req.headers["x-telegram-bot-api-secret-token"] as string;
  const expected = process.env.WEBHOOK_SECRET!;
  if (!secret || !crypto.timingSafeEqual(Buffer.from(secret), Buffer.from(expected))) {
    return res.status(403).json({ error: "Forbidden" });
  }

  await callback(req, res);
}
```

`vercel.json`:

```json
{
  "functions": {
    "api/webhook.ts": {
      "maxDuration": 10,
      "memory": 256
    }
  },
  "rewrites": [
    { "source": "/webhook", "destination": "/api/webhook" }
  ]
}
```

---

## 6. PaaS

### Railway

1. Connect GitHub repo -- Railway auto-detects Dockerfile or Nixpacks.
2. Set environment variables in Railway dashboard or CLI:

```bash
railway variables set BOT_TOKEN=123456:ABC-DEF
railway variables set WEBHOOK_SECRET=mysecret
railway up
```

For SQLite -- attach a persistent volume at `/data` and set `DATABASE_URL=sqlite:///data/bot.db`.

Railway exposes a public URL automatically. Use it as webhook URL.

### Fly.io

`fly.toml`:

```toml
app = "my-bot"
primary_region = "ams"

[build]
  dockerfile = "Dockerfile"

[env]
  LOG_LEVEL = "info"

[mounts]
  source = "botdata"
  destination = "/data"

[[services]]
  internal_port = 8080
  protocol = "tcp"
  auto_stop_machines = false    # bot must stay running
  auto_start_machines = true

  [[services.ports]]
    port = 443
    handlers = ["tls", "http"]

  [[services.http_checks]]
    interval = 15000
    grace_period = "10s"
    method = "GET"
    path = "/health"
    protocol = "http"
    timeout = 5000
```

Commands:

```bash
fly launch
fly secrets set BOT_TOKEN=123456:ABC-DEF WEBHOOK_SECRET=mysecret
fly volumes create botdata --region ams --size 1
fly deploy
fly logs
```

For polling bots (no HTTP): remove `[[services]]` and health checks.
Set `auto_stop_machines = false` -- otherwise Fly kills idle machines.

### Render

For polling bots -- use **Worker** service type (not Web Service).
Worker has no HTTP port, no health check -- just runs the process.

1. New -> Worker -> connect repo.
2. Build command: `pip install -r requirements.txt`
3. Start command: `python -m bot`
4. Environment groups: create a group `bot-secrets`, add `BOT_TOKEN`, `DATABASE_URL`.
5. Attach the group to the service.

For webhook bots -- use Web Service with a `/health` endpoint for Render health checks.

---

## 7. Setting Webhook

### Python (aiogram)

```python
import os
from aiogram import Bot

bot = Bot(token=os.environ["BOT_TOKEN"])

WEBHOOK_URL = os.environ["WEBHOOK_URL"]       # https://bot.example.com/webhook/telegram
WEBHOOK_SECRET = os.environ["WEBHOOK_SECRET"]  # random string, 1-256 chars

# Set
await bot.set_webhook(
    url=WEBHOOK_URL,
    secret_token=WEBHOOK_SECRET,
    allowed_updates=["message", "callback_query", "my_chat_member"],
    drop_pending_updates=True,
)

# Verify
info = await bot.get_webhook_info()
assert info.url == WEBHOOK_URL
assert info.has_custom_certificate is False
assert info.pending_update_count == 0

# Delete (switch back to polling)
await bot.delete_webhook(drop_pending_updates=True)
```

### TypeScript (grammY)

```typescript
const WEBHOOK_URL = process.env.WEBHOOK_URL!;
const WEBHOOK_SECRET = process.env.WEBHOOK_SECRET!;

// Set
await bot.api.setWebhook(WEBHOOK_URL, {
  secret_token: WEBHOOK_SECRET,
  allowed_updates: ["message", "callback_query", "my_chat_member"],
  drop_pending_updates: true,
});

// Verify
const info = await bot.api.getWebhookInfo();
console.log(info.url, info.pending_update_count, info.last_error_message);

// Delete
await bot.api.deleteWebhook({ drop_pending_updates: true });
```

`allowed_updates` -- always set explicitly. Reduces traffic and prevents
processing update types the bot does not handle.

`secret_token` -- Telegram sends this header with every update.
Verify it server-side. Without it, anyone who knows the webhook URL
can send fake updates.

---

## 8. Environment Variables

Standard set for any bot project:

```bash
# Required
BOT_TOKEN=123456:ABC-DEF

# Webhook mode (skip for polling)
WEBHOOK_URL=https://bot.example.com/webhook/telegram
WEBHOOK_SECRET=random-string-up-to-256-chars
WEBHOOK_PORT=8080

# Database (optional)
DATABASE_URL=postgresql://user:pass@localhost:5432/botdb

# Cache/sessions (optional)
REDIS_URL=redis://localhost:6379/0

# Access control
ADMIN_IDS=164795011,123456789

# Observability
LOG_LEVEL=INFO
SENTRY_DSN=https://key@sentry.io/123
```

Rules:
- Never commit `.env` to git. Add to `.gitignore`.
- Use `EnvironmentFile=` (systemd) or `env_file:` (Docker Compose) -- not inline env.
- `BOT_TOKEN` and `WEBHOOK_SECRET` are secrets -- store in secret managers
  (Docker secrets, Fly secrets, Railway variables, wrangler secret).
- `ADMIN_IDS` -- comma-separated Telegram user IDs. Parse at startup, not per-request.
- `LOG_LEVEL` -- standard levels: DEBUG, INFO, WARNING, ERROR.
