# AI Integration -- Telegram Bot Builder

Patterns for integrating LLMs into Telegram bots. Python (aiogram 3.x) examples.

## Table of Contents

1. OpenAI Integration (Streaming)
2. Claude / Anthropic Integration (Streaming)
3. Edit-Message Pattern (Stream to Telegram)
4. Context Management (Redis)
5. Token / Cost Management
6. Image Generation
7. Complete AI Handler Example
8. Error Handling Notes

## 1. OpenAI Integration (Streaming)

```python
import asyncio
from collections.abc import AsyncGenerator
from openai import AsyncOpenAI

client = AsyncOpenAI()  # OPENAI_API_KEY from env

async def stream_openai(
    messages: list[dict[str, str]],
    model: str = "gpt-4o",
) -> AsyncGenerator[str, None]:
    """Stream chat completion chunks."""
    stream = await client.chat.completions.create(
        model=model,
        messages=messages,
        stream=True,
    )
    async for chunk in stream:
        delta = chunk.choices[0].delta.content
        if delta:
            yield delta
```

## 2. Claude / Anthropic Integration (Streaming)

```python
from collections.abc import AsyncGenerator
from anthropic import AsyncAnthropic

anthropic = AsyncAnthropic()  # ANTHROPIC_API_KEY from env

async def stream_claude(
    messages: list[dict[str, str]],
    model: str = "claude-sonnet-4-20250514",
    system: str = "",
) -> AsyncGenerator[str, None]:
    """Stream Anthropic message chunks."""
    async with anthropic.messages.stream(
        model=model,
        max_tokens=4096,
        system=system,
        messages=messages,
    ) as stream:
        async for text in stream.text_stream:
            yield text
```

## 3. Edit-Message Pattern (Stream to Telegram)

Telegram does not support SSE. Pattern: send placeholder, then edit every N chars.

```python
from collections.abc import AsyncGenerator
from aiogram import Bot
from aiogram.types import Message

EDIT_INTERVAL_CHARS = 120
EDIT_INTERVAL_SEC = 1.5
TG_MSG_LIMIT = 4096
SPLIT_THRESHOLD = 4000  # split before hitting limit

async def stream_to_telegram(
    bot: Bot,
    chat_id: int,
    stream: AsyncGenerator[str, None],
    reply_to: int | None = None,
) -> None:
    """Send streaming LLM response with progressive edits.

    Sends a placeholder message, then edits it as chunks arrive.
    Splits into new messages when approaching the 4096 char limit.
    """
    buffer = ""
    sent_text = ""
    msg: Message | None = None
    last_edit = 0.0

    async for chunk in stream:
        buffer += chunk
        now = asyncio.get_event_loop().time()
        should_edit = (
            len(buffer) - len(sent_text) >= EDIT_INTERVAL_CHARS
            or now - last_edit >= EDIT_INTERVAL_SEC
        )

        if not should_edit:
            continue

        # first message -- send
        if msg is None:
            msg = await bot.send_message(
                chat_id,
                buffer,
                reply_to_message_id=reply_to,
            )
            sent_text = buffer
            last_edit = now
            continue

        # approaching limit -- start new message
        if len(buffer) > SPLIT_THRESHOLD:
            # finalize current message
            if buffer[:SPLIT_THRESHOLD] != sent_text:
                await _safe_edit(bot, msg, buffer[:SPLIT_THRESHOLD])
            # new message for overflow
            overflow = buffer[SPLIT_THRESHOLD:]
            buffer = overflow
            msg = await bot.send_message(chat_id, buffer)
            sent_text = buffer
            last_edit = now
            continue

        # normal edit
        if buffer != sent_text:
            await _safe_edit(bot, msg, buffer)
            sent_text = buffer
            last_edit = now

    # final edit with remaining buffer
    if msg is None and buffer:
        await bot.send_message(
            chat_id, buffer, reply_to_message_id=reply_to
        )
    elif msg and buffer != sent_text:
        await _safe_edit(bot, msg, buffer)


async def _safe_edit(bot: Bot, msg: Message, text: str) -> None:
    """Edit message, silently ignore MessageNotModified."""
    try:
        await bot.edit_message_text(
            text=text,
            chat_id=msg.chat.id,
            message_id=msg.message_id,
        )
    except Exception:
        # MessageNotModified or other Telegram errors -- skip
        pass
```

## 4. Context Management (Redis)

Store last N messages per user. Expire after 24h of inactivity.

```python
import json
import redis.asyncio as redis

HISTORY_KEY = "chat:{user_id}:history"
MAX_MESSAGES = 20
HISTORY_TTL = 86400  # 24 hours

pool = redis.ConnectionPool.from_url("redis://localhost:6379/0")


async def get_history(user_id: int) -> list[dict[str, str]]:
    """Retrieve conversation history for a user."""
    r = redis.Redis(connection_pool=pool)
    raw = await r.lrange(HISTORY_KEY.format(user_id=user_id), 0, -1)
    return [json.loads(item) for item in reversed(raw)]


async def save_message(
    user_id: int, role: str, content: str
) -> None:
    """Append a message and trim to MAX_MESSAGES."""
    r = redis.Redis(connection_pool=pool)
    key = HISTORY_KEY.format(user_id=user_id)
    await r.lpush(key, json.dumps({"role": role, "content": content}))
    await r.ltrim(key, 0, MAX_MESSAGES - 1)
    await r.expire(key, HISTORY_TTL)


async def clear_history(user_id: int) -> None:
    """Clear conversation history."""
    r = redis.Redis(connection_pool=pool)
    await r.delete(HISTORY_KEY.format(user_id=user_id))
```

## 5. Token / Cost Management

### Count tokens before sending (OpenAI)

```python
import tiktoken

def count_tokens(
    messages: list[dict[str, str]], model: str = "gpt-4o"
) -> int:
    """Count tokens in a message list using tiktoken."""
    enc = tiktoken.encoding_for_model(model)
    total = 0
    for msg in messages:
        total += 4  # every message has role/content overhead
        total += len(enc.encode(msg["content"]))
    total += 2  # assistant priming
    return total
```

### Daily usage limit per user (Redis)

```python
DAILY_LIMIT = 50  # requests per day
WARN_THRESHOLD = 0.8  # warn at 80%

async def check_usage(user_id: int) -> tuple[bool, int, bool]:
    """Check if user is within daily limit.

    Returns:
        (allowed, remaining, should_warn)
    """
    r = redis.Redis(connection_pool=pool)
    key = f"usage:{user_id}:daily"
    count = await r.incr(key)
    if count == 1:
        # first request today -- set TTL to midnight
        await r.expire(key, _seconds_until_midnight())
    remaining = max(0, DAILY_LIMIT - count)
    should_warn = count >= int(DAILY_LIMIT * WARN_THRESHOLD)
    allowed = count <= DAILY_LIMIT
    return allowed, remaining, should_warn


def _seconds_until_midnight() -> int:
    """Seconds remaining until next UTC midnight."""
    from datetime import datetime, timezone
    now = datetime.now(timezone.utc)
    return 86400 - (now.hour * 3600 + now.minute * 60 + now.second)
```

Usage in handler:

```python
allowed, remaining, should_warn = await check_usage(user_id)
if not allowed:
    await message.reply("Daily limit reached. Resets at midnight UTC.")
    return
if should_warn:
    # append warning to response later
    warning = f"\n\n-- {remaining} requests remaining today"
```

## 6. Image Generation

### /imagine command with fal.ai

```python
import os
import httpx
from aiogram.types import BufferedInputFile, Message
import redis.asyncio as redis

FAL_KEY = os.getenv("FAL_KEY")
IMG_RATE_KEY = "img:{user_id}:daily"
IMG_DAILY_LIMIT = 5


async def handle_imagine(message: Message) -> None:
    """Generate image from text prompt via fal.ai."""
    user_id = message.from_user.id
    prompt = message.text.removeprefix("/imagine").strip()
    if not prompt:
        await message.reply("Usage: /imagine <description>")
        return

    # rate limit expensive operation
    r = redis.Redis(connection_pool=pool)
    count = await r.incr(IMG_RATE_KEY.format(user_id=user_id))
    if count == 1:
        await r.expire(
            IMG_RATE_KEY.format(user_id=user_id), 86400
        )
    if count > IMG_DAILY_LIMIT:
        await message.reply(f"Image limit: {IMG_DAILY_LIMIT}/day.")
        return

    status = await message.reply("Generating...")
    await message.bot.send_chat_action(
        message.chat.id, "upload_photo"
    )

    async with httpx.AsyncClient(timeout=60.0) as http:
        resp = await http.post(
            "https://queue.fal.run/fal-ai/flux/schnell",
            headers={"Authorization": f"Key {FAL_KEY}"},
            json={
                "prompt": prompt,
                "image_size": "landscape_16_9",
                "num_images": 1,
            },
        )
        resp.raise_for_status()
        image_url = resp.json()["images"][0]["url"]

        img_resp = await http.get(image_url)
        img_resp.raise_for_status()

    photo = BufferedInputFile(
        img_resp.content, filename="generated.png"
    )
    await message.reply_photo(photo=photo)
    await status.delete()
```

### DALL-E variant

```python
import base64

async def generate_dalle(prompt: str) -> bytes:
    """Generate image via DALL-E 3, return PNG bytes."""
    response = await client.images.generate(
        model="dall-e-3",
        prompt=prompt,
        size="1024x1024",
        response_format="b64_json",
    )
    return base64.b64decode(response.data[0].b64_json)
```

## 7. Complete AI Handler Example

Full handler: typing indicator, history, stream, edit, save.

```python
from aiogram import Router, F
from aiogram.types import Message

router = Router()

SYSTEM_PROMPT = "You are a helpful assistant. Be concise."


@router.message(F.text & ~F.text.startswith("/"))
async def handle_ai_message(message: Message) -> None:
    """Process user message through LLM with streaming response."""
    user_id = message.from_user.id

    # 1. check usage limit
    allowed, remaining, should_warn = await check_usage(user_id)
    if not allowed:
        await message.reply(
            "Daily limit reached. Resets at midnight UTC."
        )
        return

    # 2. typing indicator
    await message.bot.send_chat_action(message.chat.id, "typing")

    # 3. get conversation history
    history = await get_history(user_id)
    history.append({"role": "user", "content": message.text})

    # 4. build messages for API
    messages = (
        [{"role": "system", "content": SYSTEM_PROMPT}] + history
    )

    # 5. optional: check token count, trim if needed
    tokens = count_tokens(messages)
    if tokens > 12000:
        # keep system prompt + last 6 messages
        messages = [messages[0]] + messages[-6:]

    # 6. stream response to Telegram
    full_response: list[str] = []

    async def collect_and_stream():
        async for chunk in stream_openai(messages):
            full_response.append(chunk)
            yield chunk

    await stream_to_telegram(
        bot=message.bot,
        chat_id=message.chat.id,
        stream=collect_and_stream(),
        reply_to=message.message_id,
    )

    # 7. save both messages to history
    response_text = "".join(full_response)
    await save_message(user_id, "user", message.text)
    await save_message(user_id, "assistant", response_text)

    # 8. append usage warning if needed
    if should_warn:
        await message.answer(
            f"-- {remaining} requests remaining today",
        )
```

## Error Handling Notes

- Wrap all API calls in try/except -- OpenAI/Anthropic raise on 429/500
- On rate limit (429): reply with "Too many requests, try in 30s"
- On API error: reply with generic "Something went wrong" -- never expose raw errors
- On context overflow: auto-trim history, retry once
- `MessageNotModified` from Telegram edit -- always catch and ignore
