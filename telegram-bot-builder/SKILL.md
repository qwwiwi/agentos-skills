---
name: telegram-bot-builder
description: >
  Build production-grade Telegram bots from scratch -- architecture, code,
  database, deployment, monetization. Supports Python (aiogram 3.x,
  python-telegram-bot v21) and TypeScript (grammY). Database flexibility:
  SQLite local, Supabase (PostgREST + Edge Functions + RLS), PostgreSQL
  (asyncpg, Prisma, Drizzle), Yandex Cloud, Redis. Covers webhook/polling,
  FSM, middleware, AI integration (OpenAI/Claude streaming), Telegram Stars
  payments, Mini Apps, Business API.
  Use when: (1) creating a new Telegram bot, (2) adding features to existing
  bot, (3) choosing stack/database for a bot, (4) deploying a bot to
  production, (5) integrating AI/LLM into a bot, (6) adding payments/Stars,
  (7) scaling a bot, (8) securing webhook endpoints.
  Triggers: telegram bot, tg bot, aiogram, grammy, bot api, telegram payments,
  telegram stars, bot webhook, bot polling, bot fsm, bot deployment,
  inline keyboard, conversation handler, mini app bot.
---

# Telegram Bot Builder

Build production-grade Telegram bots. This skill covers the full lifecycle:
architecture selection, framework choice, database integration, deployment,
monetization, and security.

## Minimal Hello World

**Python (aiogram 3.x):**
```python
import asyncio
from aiogram import Bot, Dispatcher, Router, F
from aiogram.types import Message

router = Router()

@router.message(F.text == "/start")
async def start(message: Message) -> None:
    await message.answer("Hello! I'm alive.")

async def main() -> None:
    bot = Bot(token="BOT_TOKEN")
    dp = Dispatcher()
    dp.include_router(router)
    await dp.start_polling(bot)

asyncio.run(main())
```

**TypeScript (grammY):**
```typescript
import { Bot } from "grammy";

const bot = new Bot(process.env.BOT_TOKEN!);

bot.command("start", (ctx) => ctx.reply("Hello! I'm alive."));

bot.start();
```

## Quick Start: Choose Your Stack

### Decision Tree

```
Q1: Language?
├── Python  → Q2
└── TypeScript/JS → grammY (see references/typescript-grammy.md)

Q2: Complexity?
├── Simple script / notification bot → python-telegram-bot v21
└── Multi-user, async, FSM, production → aiogram 3.x

Q3: Database?
├── Prototype / single instance → SQLite + WAL
├── Production / multi-instance → PostgreSQL (asyncpg or Supabase)
├── Serverless → Supabase Edge Functions + PostgREST
└── Russia (152-FZ) → Yandex Cloud (Managed PG + Cloud Functions)

Q4: Hosting?
├── Full control → VPS + systemd/PM2
├── Containers → Docker + compose
├── Serverless → Cloudflare Workers / Supabase Edge / Yandex CF
└── PaaS → Railway / Fly.io / Render
```

### Framework Comparison

| Factor | aiogram 3.x | python-telegram-bot v21 | grammY |
|--------|------------|------------------------|--------|
| Language | Python 3.10+ | Python 3.9+ | TypeScript/JS |
| Async | Native asyncio | Optional (v20+) | Native |
| FSM | StatesGroup, 5 scoping strategies | ConversationHandler | Conversations plugin (replay engine) |
| Bot API | 10.0 (May 2026) | Latest | Latest |
| Middleware | BaseMiddleware chain | Handler groups | Koa-style stack |
| Best for | Production, high-load, Russian community | Simple bots, scripts, beginners | TypeScript projects, serverless |
| Maintained | Active (3.28.2) | Active (v21.11.1) | Active |

> **Telegraf v4** is in maintenance mode (no v5 shipped). New projects should use grammY.

## Architecture Patterns

### 1. Monolith (single process)
Best for: early stage, <1000 DAU, simple logic.
```
Bot process → handles updates → DB (SQLite/Postgres)
```
Stack: aiogram + MemoryStorage + PM2/systemd.

### 2. Queue-based (webhook → queue → worker)
Best for: AI bots (LLM response >5s), high volume.
```
Telegram → webhook endpoint (200 OK immediately)
         → SQS/RabbitMQ (MessageGroupId=user_id)
         → worker process → Telegram API
```
Prevents webhook timeouts. Per-user ordering without cross-user blocking.

### 3. Serverless (cloud function per webhook)
Best for: low traffic, pay-per-use, no ops.
```
Telegram → Cloud Function → response
```
Requires: fast cold start (<1s), external state (Redis/KV/DB).
grammY has first-class serverless support: `bot.handleUpdate(req.body)`.

### 4. Microservice (separate handlers)
Best for: >100k DAU, large teams, complex domains.
Overkill for most bots. Consider queue-based first.

## Webhook vs Polling

| Scenario | Use |
|----------|-----|
| Local development | Polling -- no domain/SSL needed |
| Production VPS | Either; webhook has lower latency |
| Serverless | Webhook only -- polling needs persistent process |
| High-volume | Webhook -- 3x lower median latency |
| Behind firewall | Polling -- outbound only |

Webhook security (mandatory):
1. **Secret token**: `setWebhook(secret_token=...)` → validate `X-Telegram-Bot-Api-Secret-Token` header
2. **IP whitelist** (optional): `149.154.160.0/20`, `91.108.4.0/22`

## References (load on demand)

Load only the reference needed for the current task:

| File | When to load |
|------|-------------|
| [references/python-aiogram.md](references/python-aiogram.md) | Building with aiogram 3.x (FSM, routers, middleware, handlers) |
| [references/typescript-grammy.md](references/typescript-grammy.md) | Building with grammY (sessions, conversations, plugins) |
| [references/databases.md](references/databases.md) | Choosing/configuring database (SQLite, Supabase, PostgreSQL, Yandex Cloud) |
| [references/deployment.md](references/deployment.md) | Deploying bot (VPS, Docker, serverless, PaaS, reverse proxy) |
| [references/ai-integration.md](references/ai-integration.md) | Adding AI/LLM (streaming, context, token management) |
| [references/payments.md](references/payments.md) | Telegram Stars, Stripe, CloudPayments, subscriptions |
| [references/security.md](references/security.md) | Webhook validation, rate limiting, anti-spam, WebApp auth |

## Project Structure (aiogram 3.x)

```
bot/
├── bot.py              # Entry point, Dispatcher setup
├── config.py           # Settings from env (pydantic-settings)
├── handlers/
│   ├── __init__.py     # Router registration
│   ├── start.py        # /start, /help
│   ├── admin.py        # Admin commands
│   └── payments.py     # Payment handlers
├── middlewares/
│   ├── throttling.py   # Rate limiting
│   ├── auth.py         # Access control
│   └── i18n.py         # Internationalization
├── keyboards/
│   ├── inline.py       # InlineKeyboardMarkup builders
│   └── reply.py        # ReplyKeyboardMarkup builders
├── states/
│   └── forms.py        # StatesGroup definitions
├── services/
│   ├── db.py           # Database layer
│   └── ai.py           # AI/LLM integration
├── filters/
│   └── admin.py        # Custom filters
├── .env
├── Dockerfile
└── requirements.txt
```

## Project Structure (grammY)

```
bot/
├── src/
│   ├── bot.ts          # Bot instance, middleware registration
│   ├── config.ts       # Environment config
│   ├── handlers/
│   │   ├── start.ts
│   │   ├── admin.ts
│   │   └── payments.ts
│   ├── conversations/
│   │   └── registration.ts
│   ├── middleware/
│   │   ├── auth.ts
│   │   └── session.ts
│   ├── keyboards/
│   │   └── menus.ts
│   └── services/
│       ├── db.ts
│       └── ai.ts
├── .env
├── Dockerfile
├── package.json
└── tsconfig.json
```

## Validation Checklist

Before shipping, verify:

- [ ] Bot token in env var, never hardcoded
- [ ] `bot.catch()` or `@dp.errors()` global error handler
- [ ] Webhook: secret token validation enabled
- [ ] Rate limiting per user (Redis counter or middleware)
- [ ] Input length validation (`len(text) > MAX` → reject)
- [ ] HTML escape user input before `parse_mode="HTML"`
- [ ] FSM storage: NOT MemoryStorage in production
- [ ] Graceful shutdown: SIGINT/SIGTERM handlers
- [ ] Typing indicator (`sendChatAction("typing")`) before slow ops
- [ ] Message length: split at 4096 chars
