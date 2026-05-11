# Databases for Telegram Bots

Practical patterns for data persistence in Telegram bots.
Load this reference when choosing or configuring a database.

---

## Table of Contents

1. SQLite (Local)
2. Supabase
3. PostgreSQL (Direct)
4. Yandex Cloud
5. Redis (Sessions/Cache)
6. Scale-Based Decision Matrix

## 1. SQLite (Local)

Single-file database. Zero setup. Best for prototypes and single-instance bots.

### Python (aiosqlite)

```bash
pip install aiosqlite
```

Setup with WAL mode (concurrent reads during writes):

```python
import aiosqlite
from pathlib import Path

DB_PATH = Path("data/bot.db")


async def get_db() -> aiosqlite.Connection:
    db = await aiosqlite.connect(DB_PATH)
    await db.execute("PRAGMA journal_mode=WAL")
    await db.execute("PRAGMA busy_timeout=5000")
    await db.execute("PRAGMA synchronous=NORMAL")
    db.row_factory = aiosqlite.Row
    return db
```

CRUD pattern:

```python
async def init_db() -> None:
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("PRAGMA journal_mode=WAL")
        await db.execute("""
            CREATE TABLE IF NOT EXISTS users (
                tg_id INTEGER PRIMARY KEY,
                username TEXT,
                created_at TEXT DEFAULT (datetime('now')),
                is_active INTEGER DEFAULT 1
            )
        """)
        await db.commit()


async def add_user(tg_id: int, username: str | None) -> None:
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute(
            "INSERT OR IGNORE INTO users (tg_id, username) VALUES (?, ?)",
            (tg_id, username),
        )
        await db.commit()


async def get_user(tg_id: int) -> dict | None:
    async with aiosqlite.connect(DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute(
            "SELECT * FROM users WHERE tg_id = ?", (tg_id,)
        ) as cursor:
            row = await cursor.fetchone()
            return dict(row) if row else None


async def update_user(tg_id: int, is_active: bool) -> None:
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute(
            "UPDATE users SET is_active = ? WHERE tg_id = ?",
            (int(is_active), tg_id),
        )
        await db.commit()
```

### Node.js (better-sqlite3)

```bash
npm install better-sqlite3
```

Setup with WAL mode:

```typescript
import Database from "better-sqlite3";

const db = new Database("data/bot.db");
db.pragma("journal_mode = WAL");
db.pragma("busy_timeout = 5000");
db.pragma("synchronous = NORMAL");
```

CRUD with prepared statements:

```typescript
// Create
db.exec(`
  CREATE TABLE IF NOT EXISTS users (
    tg_id INTEGER PRIMARY KEY,
    username TEXT,
    created_at TEXT DEFAULT (datetime('now')),
    is_active INTEGER DEFAULT 1
  )
`);

// Insert
const insertUser = db.prepare(
  "INSERT OR IGNORE INTO users (tg_id, username) VALUES (?, ?)"
);
insertUser.run(123456, "johndoe");

// Select
const getUser = db.prepare("SELECT * FROM users WHERE tg_id = ?");
const user = getUser.get(123456);

// Update
const deactivateUser = db.prepare(
  "UPDATE users SET is_active = 0 WHERE tg_id = ?"
);
deactivateUser.run(123456);

// Batch insert (transaction -- 100x faster than individual inserts)
const insertMany = db.transaction((users: { tgId: number; username: string }[]) => {
  for (const u of users) {
    insertUser.run(u.tgId, u.username);
  }
});
insertMany([{ tgId: 1, username: "a" }, { tgId: 2, username: "b" }]);
```

### When to Upgrade from SQLite

Upgrade when any of these apply:

- **>100 concurrent writers** -- WAL helps, but write lock is still global
- **Multiple bot instances** (horizontal scaling) -- file locking breaks across containers
- **Need LISTEN/NOTIFY** -- event-driven updates (payment confirmed, status changed)
- **Need full-text search at scale** -- SQLite FTS5 works but PostgreSQL tsvector is stronger
- **Need row-level security** -- Supabase RLS or PostgreSQL policies
- **Data exceeds 1 GB** -- SQLite works but backup/replication gets painful

---

## 2. Supabase

Managed PostgreSQL + PostgREST API + Auth + Edge Functions.
Best for: rapid development, built-in auth, realtime, serverless-friendly.

### Python Client (supabase-py)

```bash
pip install supabase
```

Init with service role key (NEVER anon key on server-side bot):

```python
from supabase import create_client

SUPABASE_URL = os.environ["SUPABASE_URL"]
SUPABASE_SERVICE_KEY = os.environ["SUPABASE_SERVICE_KEY"]  # service_role, not anon

supabase = create_client(SUPABASE_URL, SUPABASE_SERVICE_KEY)
```

CRUD operations:

```python
# Insert
supabase.table("users").insert({
    "tg_id": 123456,
    "username": "johndoe",
    "email": "john@example.com",
}).execute()

# Select
result = supabase.table("users").select("*").eq("tg_id", 123456).execute()
user = result.data[0] if result.data else None

# Update
supabase.table("users").update({
    "is_active": False,
}).eq("tg_id", 123456).execute()

# Delete
supabase.table("users").delete().eq("tg_id", 123456).execute()

# Upsert (insert or update on conflict)
supabase.table("users").upsert({
    "tg_id": 123456,
    "username": "johndoe_updated",
}).execute()

# Filter queries
result = (
    supabase.table("payments")
    .select("*")
    .eq("status", "paid")
    .in_("wave", ["wave1", "wave2"])
    .gte("amount", 1000)
    .order("created_at", desc=True)
    .limit(50)
    .execute()
)
```

### TypeScript Client (@supabase/supabase-js)

```bash
npm install @supabase/supabase-js
```

```typescript
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_KEY! // service_role, not anon
);

// Insert
await supabase.from("users").insert({
  tg_id: 123456,
  username: "johndoe",
});

// Select
const { data: user } = await supabase
  .from("users")
  .select("*")
  .eq("tg_id", 123456)
  .single();

// Update
await supabase
  .from("users")
  .update({ is_active: false })
  .eq("tg_id", 123456);

// Delete
await supabase.from("users").delete().eq("tg_id", 123456);

// Upsert
await supabase.from("users").upsert({
  tg_id: 123456,
  username: "johndoe_updated",
});

// Filters
const { data: payments } = await supabase
  .from("payments")
  .select("*")
  .eq("status", "paid")
  .in("wave", ["wave1", "wave2"])
  .gte("amount", 1000)
  .order("created_at", { ascending: false })
  .limit(50);
```

### RLS Patterns

**Service role bypasses RLS.** Use it for bot backends -- the bot is a trusted server process.

```sql
-- Table with RLS enabled
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Policy for authenticated users (frontend/app)
CREATE POLICY "users_own_data" ON users
  FOR ALL USING (auth.uid()::text = user_id);

-- No anon policies needed for bot-only tables.
-- Service role key bypasses ALL policies.
```

When to use anon key + RLS:

- User-facing frontends (Mini Apps, web dashboards)
- Edge Functions that handle untrusted user requests
- NEVER on server-side bot processes -- use service role

### Edge Functions (Deno)

Webhook handler for Telegram bot on Supabase Edge Functions:

```typescript
// supabase/functions/telegram-webhook/index.ts
import { serve } from "https://deno.land/std@0.168.0/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const BOT_TOKEN = Deno.env.get("BOT_TOKEN")!;
const WEBHOOK_SECRET = Deno.env.get("WEBHOOK_SECRET")!;

serve(async (req) => {
  // Validate secret token
  const secret = req.headers.get("x-telegram-bot-api-secret-token");
  if (secret !== WEBHOOK_SECRET) {
    return new Response("Unauthorized", { status: 401 });
  }

  const update = await req.json();
  const message = update.message;
  if (!message?.text) {
    return new Response("OK");
  }

  // Supabase client inside function
  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
  );

  // Save user
  await supabase.from("users").upsert({
    tg_id: message.from.id,
    username: message.from.username,
  });

  // Respond
  await fetch(
    `https://api.telegram.org/bot${BOT_TOKEN}/sendMessage`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        chat_id: message.chat.id,
        text: `Hello, ${message.from.first_name}!`,
      }),
    }
  );

  return new Response("OK");
});
```

Limits (as of May 2026):

| Limit | Value |
|-------|-------|
| CPU time | 2 seconds |
| Memory | 256 MB |
| Wall clock | 400 seconds |
| Payload | 6 MB |
| Concurrent | 200 per function |

For AI/LLM bots (response >2s CPU): return 200 immediately, process via
database trigger or background queue.

### Realtime Subscriptions

Subscribe to table changes from bot process -- react to external events:

```python
# Use case: payment confirmed in dashboard -> bot sends message
import asyncio
from supabase import create_client

supabase = create_client(SUPABASE_URL, SUPABASE_SERVICE_KEY)


def on_payment_confirmed(payload: dict) -> None:
    """Called when payment status changes to 'paid'."""
    record = payload["record"]
    tg_id = record["tg_id"]
    amount = record["amount"]
    # Send Telegram message via bot API
    send_telegram_message(tg_id, f"Payment of {amount} confirmed!")


supabase.table("payments").on("UPDATE", on_payment_confirmed).subscribe()
```

```typescript
// TypeScript equivalent
const channel = supabase
  .channel("payment-updates")
  .on(
    "postgres_changes",
    {
      event: "UPDATE",
      schema: "public",
      table: "payments",
      filter: "status=eq.paid",
    },
    (payload) => {
      const { tg_id, amount } = payload.new;
      sendTelegramMessage(tg_id, `Payment of ${amount} confirmed!`);
    }
  )
  .subscribe();
```

---

## 3. PostgreSQL (Direct)

Full-featured relational database. Best for production bots needing
transactions, LISTEN/NOTIFY, advanced queries.

### Python (asyncpg)

```bash
pip install asyncpg
```

Connection pool setup:

```python
import asyncpg

pool: asyncpg.Pool | None = None

DATABASE_URL = os.environ["DATABASE_URL"]


async def init_pool() -> None:
    global pool
    pool = await asyncpg.create_pool(
        DATABASE_URL,
        min_size=2,
        max_size=10,
        command_timeout=30,
    )


async def close_pool() -> None:
    if pool:
        await pool.close()
```

CRUD with parameterized queries:

```python
async def create_tables() -> None:
    async with pool.acquire() as conn:
        await conn.execute("""
            CREATE TABLE IF NOT EXISTS users (
                tg_id BIGINT PRIMARY KEY,
                username TEXT,
                created_at TIMESTAMPTZ DEFAULT now(),
                is_active BOOLEAN DEFAULT true
            )
        """)


async def add_user(tg_id: int, username: str | None) -> None:
    async with pool.acquire() as conn:
        await conn.execute(
            """
            INSERT INTO users (tg_id, username)
            VALUES ($1, $2)
            ON CONFLICT (tg_id) DO UPDATE SET username = $2
            """,
            tg_id,
            username,
        )


async def get_user(tg_id: int) -> dict | None:
    async with pool.acquire() as conn:
        row = await conn.fetchrow(
            "SELECT * FROM users WHERE tg_id = $1", tg_id
        )
        return dict(row) if row else None


async def get_active_users(limit: int = 100) -> list[dict]:
    async with pool.acquire() as conn:
        rows = await conn.fetch(
            "SELECT * FROM users WHERE is_active = true LIMIT $1",
            limit,
        )
        return [dict(r) for r in rows]
```

LISTEN/NOTIFY for event-driven updates:

```python
async def listen_payments() -> None:
    """Subscribe to payment events via PostgreSQL LISTEN/NOTIFY."""
    conn = await asyncpg.connect(DATABASE_URL)
    await conn.add_listener("payment_confirmed", on_payment_notify)
    # Keep connection alive
    while True:
        await asyncio.sleep(3600)


def on_payment_notify(
    conn: asyncpg.Connection,
    pid: int,
    channel: str,
    payload: str,
) -> None:
    data = json.loads(payload)
    tg_id = data["tg_id"]
    amount = data["amount"]
    asyncio.create_task(send_payment_message(tg_id, amount))


# Trigger on the DB side:
# CREATE OR REPLACE FUNCTION notify_payment() RETURNS trigger AS $$
# BEGIN
#   PERFORM pg_notify('payment_confirmed',
#     json_build_object('tg_id', NEW.tg_id, 'amount', NEW.amount)::text);
#   RETURN NEW;
# END;
# $$ LANGUAGE plpgsql;
#
# CREATE TRIGGER payment_trigger AFTER UPDATE ON payments
#   FOR EACH ROW WHEN (NEW.status = 'paid' AND OLD.status != 'paid')
#   EXECUTE FUNCTION notify_payment();
```

**WARNING: asyncpg + PgBouncer Transaction mode = broken.**
asyncpg uses prepared statements by default. PgBouncer in transaction
mode does not support named prepared statements (different connections
on each query). Fix options:

1. Use PgBouncer session mode (defeats pooling purpose)
2. Use `statement_cache_size=0` in asyncpg connect:
   ```python
   pool = await asyncpg.create_pool(DSN, statement_cache_size=0)
   ```
3. Switch to psycopg3 (no prepared statements by default)

### Node.js (Drizzle ORM)

```bash
npm install drizzle-orm postgres
npm install -D drizzle-kit
```

Schema definition:

```typescript
// src/db/schema.ts
import { pgTable, bigint, text, boolean, timestamp } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  tgId: bigint("tg_id", { mode: "number" }).primaryKey(),
  username: text("username"),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow(),
  isActive: boolean("is_active").default(true),
});

export const payments = pgTable("payments", {
  id: bigint("id", { mode: "number" }).primaryKey().generatedAlwaysAsIdentity(),
  tgId: bigint("tg_id", { mode: "number" }).references(() => users.tgId),
  amount: bigint("amount", { mode: "number" }).notNull(),
  status: text("status").default("pending"),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow(),
});
```

CRUD operations:

```typescript
// src/db/index.ts
import { drizzle } from "drizzle-orm/postgres-js";
import postgres from "postgres";
import { eq, and, gte, inArray } from "drizzle-orm";
import * as schema from "./schema";

const client = postgres(process.env.DATABASE_URL!);
const db = drizzle(client, { schema });

// Insert
await db.insert(schema.users).values({
  tgId: 123456,
  username: "johndoe",
}).onConflictDoUpdate({
  target: schema.users.tgId,
  set: { username: "johndoe" },
});

// Select
const user = await db.query.users.findFirst({
  where: eq(schema.users.tgId, 123456),
});

// Update
await db
  .update(schema.users)
  .set({ isActive: false })
  .where(eq(schema.users.tgId, 123456));

// Delete
await db.delete(schema.users).where(eq(schema.users.tgId, 123456));

// Complex filter
const paidPayments = await db
  .select()
  .from(schema.payments)
  .where(
    and(
      eq(schema.payments.status, "paid"),
      gte(schema.payments.amount, 1000)
    )
  )
  .limit(50);
```

Migrations:

```bash
# Generate migration from schema changes
npx drizzle-kit generate

# Apply migrations
npx drizzle-kit migrate

# drizzle.config.ts
# export default {
#   schema: "./src/db/schema.ts",
#   out: "./drizzle",
#   dialect: "postgresql",
#   dbCredentials: { url: process.env.DATABASE_URL! },
# };
```

### Managed Options

| Provider | Best for | Free tier | Notes |
|----------|----------|-----------|-------|
| **Supabase** | Full-stack (auth, realtime, storage) | 500 MB, 2 projects | PostgREST API, Edge Functions, RLS |
| **Neon** | Serverless, branching | 512 MB, 1 project | Auto-suspend, database branching for CI |
| **Railway** | Simple deploy, good DX | $5 credit/month | Persistent, no auto-suspend |
| **Timescale** | Time-series data, analytics bots | 25 GB compressed | Hypertables, continuous aggregates |

---

## 4. Yandex Cloud

Russian infrastructure. Relevant for 152-FZ compliance (personal data
of Russian citizens stored in Russia).

### Cloud Functions Webhook Handler

```python
# index.py -- Yandex Cloud Function entry point
import json
import os

import psycopg2
import requests

BOT_TOKEN = os.environ["BOT_TOKEN"]
DB_HOST = os.environ["DB_HOST"]  # Managed PostgreSQL internal hostname
DB_NAME = os.environ["DB_NAME"]
DB_USER = os.environ["DB_USER"]
DB_PASSWORD = os.environ["DB_PASSWORD"]


def handler(event: dict, context) -> dict:
    """Yandex Cloud Function handler for Telegram webhook."""
    body = json.loads(event.get("body", "{}"))
    message = body.get("message")
    if not message or not message.get("text"):
        return {"statusCode": 200, "body": "OK"}

    tg_id = message["from"]["id"]
    username = message["from"].get("username")
    text = message["text"]

    # Connect to Managed PostgreSQL
    conn = psycopg2.connect(
        host=DB_HOST,
        dbname=DB_NAME,
        user=DB_USER,
        password=DB_PASSWORD,
        sslmode="verify-full",
    )
    cur = conn.cursor()
    cur.execute(
        """
        INSERT INTO users (tg_id, username)
        VALUES (%s, %s)
        ON CONFLICT (tg_id) DO UPDATE SET username = %s
        """,
        (tg_id, username, username),
    )
    conn.commit()
    cur.close()
    conn.close()

    # Reply
    requests.post(
        f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage",
        json={"chat_id": message["chat"]["id"], "text": f"Got: {text}"},
    )

    return {"statusCode": 200, "body": "OK"}
```

Store bot token as environment variable in Cloud Function settings,
never in code. Use Yandex Lockbox for production secrets.

### Pricing (as of May 2026)

| Resource | Free tier | Paid |
|----------|-----------|------|
| Invocations | 1,000,000/month | 10 RUB per 1M |
| Compute (GB*s) | 10 GB*s/month | ~16 RUB per GB*hour |
| Managed PostgreSQL | None | ~4 RUB/hour (s2.micro) |

Formula: `cost = invocations * 0.00001 + gb_seconds * 0.0000016 + egress_gb * 0.12`

For bots under 1M messages/month, Cloud Functions cost is near zero.
The database is the main expense.

---

## 5. Redis (Sessions/Cache)

In-memory store. Use for FSM state, rate limiting, caching.
NOT a primary database -- data is ephemeral (unless persistence enabled).

### aiogram RedisStorage

```bash
pip install aiogram[redis]
# or: pip install redis
```

```python
from aiogram.fsm.storage.redis import RedisStorage
from redis.asyncio import Redis

REDIS_URL = os.environ.get("REDIS_URL", "redis://localhost:6379/0")

redis = Redis.from_url(REDIS_URL)
storage = RedisStorage(redis)

# State TTL -- expire inactive conversations after 24 hours
storage = RedisStorage(
    redis,
    state_ttl=86400,   # 24h for FSM state
    data_ttl=86400,    # 24h for FSM data
)
```

Key format (aiogram default):
```
fsm:{bot_id}:{chat_id}:{user_id}:state   -> "FormStates:waiting_email"
fsm:{bot_id}:{chat_id}:{user_id}:data    -> {"name": "John", "step": 2}
```

### grammY Redis Sessions

```bash
npm install @grammyjs/storage-redis ioredis
```

```typescript
import { Bot, session } from "grammy";
import { RedisAdapter } from "@grammyjs/storage-redis";
import { Redis } from "ioredis";

interface SessionData {
  step: string;
  formData: Record<string, string>;
}

const redis = new Redis(process.env.REDIS_URL);

const bot = new Bot(process.env.BOT_TOKEN!);

bot.use(
  session({
    initial: (): SessionData => ({ step: "idle", formData: {} }),
    storage: new RedisAdapter({ instance: redis }),
  })
);
```

### When Redis is Needed vs Not

| Scenario | Redis needed? | Alternative |
|----------|--------------|-------------|
| Multi-instance bot (2+ processes) | Yes -- shared state mandatory | -- |
| Single instance, simple FSM | No | `MemoryStorage` (aiogram) or in-memory session (grammY) |
| Single instance, persistent FSM | No | SQLite-backed storage |
| Rate limiting (multi-instance) | Yes | -- |
| Rate limiting (single instance) | No | In-memory counter dict |
| Caching API responses | Yes (if shared) | `functools.lru_cache` or TTLCache |
| Job queue | Consider | `asyncio.Queue` for single instance |

Rule of thumb: if you run 1 process and can tolerate state loss on
restart -- skip Redis. Add it when scaling to multiple instances.

---

## 6. Scale-Based Decision Matrix

| Scale | DB | FSM Storage | Hosting | Notes |
|-------|-----|------------|---------|-------|
| **Hobby** (<100 users) | SQLite | MemoryStorage | VPS $5/mo, systemd | Simplest. Backup: cron + cp |
| **Startup** (100-10k users) | Supabase free | Supabase (table) or Redis | VPS or Railway | PostgREST saves dev time |
| **Production** (10k-100k users) | PostgreSQL (managed) | Redis | VPS + Docker Compose | Connection pool mandatory |
| **Serverless** | Supabase or Neon | KV store (Cloudflare) or Redis | Edge Functions or CF Workers | Cold start matters -- keep deps minimal |
| **Russia compliance** (152-FZ) | Yandex Managed PG | Redis (Yandex) | Yandex Cloud Functions or VPS in RU | PD of Russian citizens must be stored in Russia |

Key points:

- Start with SQLite. Migrate when you hit the limits (see section 1).
- Supabase free tier covers most bots up to 10k users.
- PostgreSQL direct connection when you need LISTEN/NOTIFY, complex
  transactions, or outgrow PostgREST.
- Redis is for shared ephemeral state, not primary storage.
- 152-FZ: if bot collects name + phone/email of Russian citizens,
  primary storage must be in Russia. Supabase (eu-west-1) does not qualify.
