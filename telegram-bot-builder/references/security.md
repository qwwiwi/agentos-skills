# Security -- Telegram Bot Builder

Webhook validation, rate limiting, input sanitization, WebApp auth, anti-spam.

## Table of Contents

1. Webhook Validation
2. Rate Limiting
3. Input Sanitization
4. WebApp (Mini App) Data Validation
5. Anti-Spam
6. Bot Token Security

## 1. Webhook Validation

### Secret token setup and validation (Python / FastAPI)

```python
import os
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()

WEBHOOK_SECRET = os.getenv("TELEGRAM_WEBHOOK_SECRET")
# set during setWebhook:
# await bot.set_webhook(url=URL, secret_token=WEBHOOK_SECRET)


@app.post("/webhook")
async def webhook(request: Request) -> dict:
    """Receive Telegram updates with secret token validation."""
    token = request.headers.get("X-Telegram-Bot-Api-Secret-Token")
    if token != WEBHOOK_SECRET:
        raise HTTPException(status_code=403, detail="Forbidden")

    update = await request.json()
    await dp.feed_webhook_update(bot, update)
    return {"ok": True}
```

### Secret token validation (TypeScript / Express)

```typescript
import express from "express";

const WEBHOOK_SECRET = process.env.TELEGRAM_WEBHOOK_SECRET!;

app.post("/webhook", (req, res) => {
  const token = req.headers["x-telegram-bot-api-secret-token"];
  if (token !== WEBHOOK_SECRET) {
    res.status(403).send("Forbidden");
    return;
  }
  // process update
  bot.handleUpdate(req.body);
  res.sendStatus(200);
});
```

### IP whitelist (optional, additional layer)

Telegram sends webhooks from these subnets only:

```python
from ipaddress import ip_address, ip_network

TELEGRAM_SUBNETS = [
    ip_network("149.154.160.0/20"),
    ip_network("91.108.4.0/22"),
]


def is_telegram_ip(ip: str) -> bool:
    """Check if IP belongs to Telegram's webhook subnets."""
    addr = ip_address(ip)
    return any(addr in subnet for subnet in TELEGRAM_SUBNETS)
```

Use with reverse proxy: check `X-Forwarded-For` or `X-Real-IP`.
Caddy/nginx must be configured to set these headers from the real
client IP, not from the proxy itself.

## 2. Rate Limiting

### Per-user Redis counter

```python
import redis.asyncio as redis

pool = redis.ConnectionPool.from_url("redis://localhost:6379/0")

RATE_WINDOW = 60  # seconds
RATE_LIMIT = 20  # messages per window


async def is_rate_limited(user_id: int) -> bool:
    """Check if user exceeds rate limit. Returns True if blocked."""
    r = redis.Redis(connection_pool=pool)
    key = f"rate:{user_id}"
    count = await r.incr(key)
    if count == 1:
        await r.expire(key, RATE_WINDOW)
    return count > RATE_LIMIT
```

### aiogram ThrottlingMiddleware

```python
from aiogram import BaseMiddleware
from aiogram.types import TelegramObject, Message


class ThrottlingMiddleware(BaseMiddleware):
    """Rate limit per user using Redis."""

    def __init__(
        self,
        rate_limit: int = 20,
        window: int = 60,
    ) -> None:
        self.rate_limit = rate_limit
        self.window = window

    async def __call__(
        self,
        handler,
        event: TelegramObject,
        data: dict,
    ):
        if not isinstance(event, Message):
            return await handler(event, data)

        user_id = event.from_user.id
        if await is_rate_limited(user_id):
            await event.reply("Too many requests. Wait a minute.")
            return

        return await handler(event, data)
```

### Telegram API limits (hard limits, not configurable)

| Scope | Limit |
|-------|-------|
| Messages to same chat | 1 msg/second |
| Messages to different chats | 30 msg/second |
| Bulk notifications | 30 msg/second, 429 after |
| Inline query results | 50 results max |
| Group: bot messages | 20 msg/minute per group |

For bulk sends, use a queue with 30 msg/s throttle and exponential
backoff on 429 errors.

## 3. Input Sanitization

### HTML escape before parse_mode="HTML"

```python
import html


async def safe_reply_html(
    message, template: str, **kwargs: str
) -> None:
    """Reply with HTML, escaping all user-provided values."""
    safe_kwargs = {k: html.escape(str(v)) for k, v in kwargs.items()}
    text = template.format(**safe_kwargs)
    await message.reply(text, parse_mode="HTML")


# usage:
await safe_reply_html(
    message,
    "Hello, <b>{name}</b>! Your query: <code>{query}</code>",
    name=message.from_user.full_name,
    query=user_input,
)
```

### Input length validation

```python
MAX_INPUT_LENGTH = 4000  # chars
MAX_COMMAND_ARGS = 500


async def validate_input(message: Message) -> str | None:
    """Validate and return cleaned input, or None if invalid."""
    text = message.text
    if not text:
        return None

    if len(text) > MAX_INPUT_LENGTH:
        await message.reply(
            f"Message too long (max {MAX_INPUT_LENGTH} chars)."
        )
        return None

    return text.strip()
```

### Reject unexpected content types

```python
from aiogram import Router, F
from aiogram.types import Message

router = Router()


@router.message(~F.text & ~F.photo & ~F.document)
async def reject_unexpected(message: Message) -> None:
    """Reject unsupported content types."""
    await message.reply(
        "I only accept text messages, photos, and documents."
    )
```

## 4. WebApp (Mini App) Data Validation

Telegram sends `initData` signed with bot token. Validate before trusting.

### Python: HMAC-SHA256 validation

```python
import hashlib
import hmac
import json
from urllib.parse import parse_qs, unquote


def validate_webapp_data(
    init_data: str, bot_token: str
) -> dict | None:
    """Validate Telegram WebApp initData.

    Returns parsed data dict if valid, None if invalid.
    """
    parsed = parse_qs(init_data)
    received_hash = parsed.get("hash", [None])[0]
    if not received_hash:
        return None

    # build data-check-string: sorted key=value pairs without hash
    pairs = []
    for key_val in init_data.split("&"):
        key, _, value = key_val.partition("=")
        if key != "hash":
            pairs.append(f"{key}={unquote(value)}")
    pairs.sort()
    data_check_string = "\n".join(pairs)

    # HMAC chain: secret_key = HMAC-SHA256("WebAppData", bot_token)
    secret_key = hmac.new(
        b"WebAppData", bot_token.encode(), hashlib.sha256
    ).digest()
    computed_hash = hmac.new(
        secret_key, data_check_string.encode(), hashlib.sha256
    ).hexdigest()

    if not hmac.compare_digest(computed_hash, received_hash):
        return None

    # parse user data
    result = {}
    for key, values in parsed.items():
        value = values[0]
        if key == "user":
            result["user"] = json.loads(unquote(value))
        else:
            result[key] = unquote(value)
    return result
```

### TypeScript equivalent

```typescript
import crypto from "crypto";

interface WebAppUser {
  id: number;
  first_name: string;
  last_name?: string;
  username?: string;
}

function validateWebAppData(
  initData: string,
  botToken: string
): WebAppUser | null {
  const params = new URLSearchParams(initData);
  const hash = params.get("hash");
  if (!hash) return null;

  // build data-check-string
  const pairs: string[] = [];
  params.forEach((value, key) => {
    if (key !== "hash") pairs.push(`${key}=${value}`);
  });
  pairs.sort();
  const dataCheckString = pairs.join("\n");

  // HMAC chain
  const secretKey = crypto
    .createHmac("sha256", "WebAppData")
    .update(botToken)
    .digest();
  const computedHash = crypto
    .createHmac("sha256", secretKey)
    .update(dataCheckString)
    .digest("hex");

  if (
    !crypto.timingSafeEqual(
      Buffer.from(computedHash),
      Buffer.from(hash)
    )
  ) {
    return null;
  }

  const userStr = params.get("user");
  if (!userStr) return null;
  return JSON.parse(userStr) as WebAppUser;
}
```

## 5. Anti-Spam

### Shadow ban pattern

User does not know they are banned. Bot silently ignores their messages.

```python
SHADOW_BANNED: set[int] = set()
# in production, store in Redis: SISMEMBER shadow_ban {user_id}


class ShadowBanMiddleware(BaseMiddleware):
    """Silently drop messages from shadow-banned users."""

    async def __call__(
        self,
        handler,
        event: TelegramObject,
        data: dict,
    ):
        if isinstance(event, Message):
            if event.from_user.id in SHADOW_BANNED:
                return  # silently ignore
        return await handler(event, data)
```

### CAPTCHA on /start

```python
import secrets

CAPTCHA_STORE: dict[int, str] = {}  # user_id -> expected answer

@router.message(F.text == "/start")
async def captcha_start(message: Message) -> None:
    """Send simple math CAPTCHA before allowing bot usage."""
    a = secrets.randbelow(10) + 1
    b = secrets.randbelow(10) + 1
    answer = str(a + b)
    CAPTCHA_STORE[message.from_user.id] = answer
    await message.reply(f"Solve to continue: {a} + {b} = ?")


@router.message(F.text)
async def captcha_check(message: Message) -> None:
    """Verify CAPTCHA answer."""
    user_id = message.from_user.id
    expected = CAPTCHA_STORE.get(user_id)
    if expected is None:
        # already verified or no CAPTCHA needed
        return

    if message.text.strip() == expected:
        del CAPTCHA_STORE[user_id]
        await message.reply("Verified! Welcome.")
    else:
        await message.reply("Wrong answer. Try /start again.")
```

### CAS (Combot Anti-Spam) API check

```python
import httpx

CAS_API = "https://api.cas.chat/check"


async def is_cas_banned(user_id: int) -> bool:
    """Check if user is in CAS ban database."""
    async with httpx.AsyncClient(timeout=5.0) as http:
        try:
            resp = await http.get(
                CAS_API, params={"user_id": user_id}
            )
            data = resp.json()
            return data.get("ok", False)
        except Exception:
            return False  # fail open -- do not block on API errors
```

### User scoring + auto-ban

```python
async def score_new_user(user_id: int, user) -> int:
    """Score a new user. Higher = more suspicious.

    Returns:
        Score 0-100. Ban threshold: 80.
    """
    score = 0

    # no username
    if not user.username:
        score += 20

    # no profile photo (requires getUserProfilePhotos)
    # photos = await bot.get_user_profile_photos(user_id, limit=1)
    # if photos.total_count == 0:
    #     score += 15

    # very new account (low user_id is old, high is new)
    # approximate: IDs above 6B are from 2023+
    if user_id > 7_000_000_000:
        score += 15

    # CAS banned
    if await is_cas_banned(user_id):
        score += 50

    # name contains spam patterns
    name = (user.first_name or "") + " " + (user.last_name or "")
    spam_patterns = ["free", "earn", "crypto", "invest", "click"]
    if any(p in name.lower() for p in spam_patterns):
        score += 25

    return min(score, 100)
```

## 6. Bot Token Security

### Never hardcode

```python
# BAD
TOKEN = "123456:ABC-DEF..."

# GOOD
import os
TOKEN = os.getenv("BOT_TOKEN")
if not TOKEN:
    raise RuntimeError("BOT_TOKEN env var is required")
```

### pydantic-settings pattern (recommended)

```python
from pydantic_settings import BaseSettings


class BotConfig(BaseSettings):
    """Bot configuration from environment variables."""

    bot_token: str
    webhook_secret: str
    redis_url: str = "redis://localhost:6379/0"
    admin_ids: list[int] = []

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}


config = BotConfig()
```

### Rotate token via @BotFather

1. `/mybots` -> select bot -> API Token -> Revoke
2. Copy new token
3. Update `.env` on all servers
4. Restart bot process

Old token stops working immediately. No grace period.

### Separate tokens for dev/staging/prod

Create 3 bots in @BotFather:
- `@MyBot` -- production
- `@MyBotStaging` -- staging
- `@MyBotDev` -- development

Each has its own token. Never use production token in development.

### .env security

```bash
# file permissions
chmod 600 .env

# .gitignore -- always include
echo ".env" >> .gitignore
echo "*.key" >> .gitignore
echo "*.pem" >> .gitignore

# docker -- use secrets, not env vars for sensitive data
# docker-compose.yml:
# secrets:
#   bot_token:
#     file: ./secrets/bot_token.txt
```

## Security Checklist

- [ ] Bot token in env var, never in code
- [ ] Webhook: secret token validation enabled
- [ ] Webhook: HTTPS only (Telegram requirement)
- [ ] Input: html.escape() before parse_mode="HTML"
- [ ] Input: length validation on all text inputs
- [ ] Rate limiting: per-user, per-endpoint
- [ ] Admin commands: check user_id against allowlist
- [ ] WebApp: HMAC validation of initData on server
- [ ] Error messages: never expose stack traces or internal details
- [ ] Logging: never log bot token or user passwords
- [ ] .env in .gitignore
- [ ] Separate bot tokens for dev/staging/prod
