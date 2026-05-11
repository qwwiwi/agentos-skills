# aiogram 3.x -- Code Patterns

Python 3.10+. All examples use aiogram 3.28+.

---

## Table of Contents

1. Bot Setup
2. Routers
3. Handlers
4. FSM (Finite State Machine)
5. Middleware
6. Keyboards
7. Magic Filters
8. Dependency Injection
9. Entry Point
10. Quick Reference: pip install


## 1. Bot Setup

### Polling

```python
import asyncio
import logging

from aiogram import Bot, Dispatcher
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode

from handlers import router as main_router

logging.basicConfig(level=logging.INFO)


async def main() -> None:
    bot = Bot(
        token="BOT_TOKEN",
        default=DefaultBotProperties(parse_mode=ParseMode.HTML),
    )
    dp = Dispatcher()
    dp.include_router(main_router)

    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
```

### Webhook (FastAPI)

```python
import logging
from contextlib import asynccontextmanager
from typing import AsyncGenerator

from aiogram import Bot, Dispatcher
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.types import Update
from fastapi import FastAPI, Header, HTTPException, Request

WEBHOOK_PATH = "/webhook/telegram"
WEBHOOK_SECRET = "random-secret-string-here"
BASE_URL = "https://example.com"

bot = Bot(
    token="BOT_TOKEN",
    default=DefaultBotProperties(parse_mode=ParseMode.HTML),
)
dp = Dispatcher()


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    await bot.set_webhook(
        url=f"{BASE_URL}{WEBHOOK_PATH}",
        secret_token=WEBHOOK_SECRET,
    )
    yield
    await bot.delete_webhook()
    await bot.session.close()


app = FastAPI(lifespan=lifespan)


@app.post(WEBHOOK_PATH)
async def telegram_webhook(
    request: Request,
    x_telegram_bot_api_secret_token: str = Header(None),
) -> dict[str, bool]:
    if x_telegram_bot_api_secret_token != WEBHOOK_SECRET:
        raise HTTPException(status_code=403, detail="Forbidden")

    update = Update.model_validate(await request.json(), context={"bot": bot})
    await dp.feed_update(bot, update)
    return {"ok": True}
```

Run: `uvicorn bot:app --host 0.0.0.0 --port 8080`

### Config from env (pydantic-settings)

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
    )

    bot_token: str
    webhook_secret: str = ""
    webhook_base_url: str = ""
    admin_ids: list[int] = []
    redis_url: str = "redis://localhost:6379/0"
    database_url: str = "sqlite+aiosqlite:///bot.db"
    debug: bool = False


settings = Settings()
```

`.env`:
```
BOT_TOKEN=123456:ABC-DEF
ADMIN_IDS=[164795011,999999]
REDIS_URL=redis://localhost:6379/0
```

---

## 2. Routers

### Creating routers

```python
# handlers/start.py
from aiogram import Router
from aiogram.filters import CommandStart
from aiogram.types import Message

router = Router(name="start")


@router.message(CommandStart())
async def cmd_start(message: Message) -> None:
    await message.answer(f"Hello, {message.from_user.full_name}!")
```

### Including sub-routers

```python
# handlers/__init__.py
from aiogram import Router

from .admin import router as admin_router
from .payments import router as payments_router
from .start import router as start_router


def setup_routers() -> Router:
    root = Router(name="root")
    root.include_routers(
        admin_router,   # checked first -- order matters
        payments_router,
        start_router,
    )
    return root
```

```python
# bot.py
dp = Dispatcher()
dp.include_router(setup_routers())
```

### Router-level filters

```python
from aiogram import F, Router

# Only private chats -- all handlers in this router inherit the filter
private_router = Router(name="private")
private_router.message.filter(F.chat.type == "private")

# Only specific group
group_router = Router(name="group")
group_router.message.filter(F.chat.id == -1001234567890)
```

---

## 3. Handlers

### Command handlers

```python
from aiogram import Router
from aiogram.filters import Command, CommandObject, CommandStart
from aiogram.types import Message

router = Router()


@router.message(CommandStart())
async def cmd_start(message: Message) -> None:
    await message.answer("Welcome!")


# /start with deep link: /start ref_12345
@router.message(CommandStart(deep_link=True))
async def cmd_start_deep(message: Message, command: CommandObject) -> None:
    ref = command.args  # "ref_12345"
    await message.answer(f"Referred by: {ref}")


@router.message(Command("help"))
async def cmd_help(message: Message) -> None:
    await message.answer("Available commands: /start, /help, /settings")


# Command with arguments: /ban 12345 spam
@router.message(Command("ban"))
async def cmd_ban(message: Message, command: CommandObject) -> None:
    if not command.args:
        await message.answer("Usage: /ban <user_id> <reason>")
        return
    parts = command.args.split(maxsplit=1)
    user_id = int(parts[0])
    reason = parts[1] if len(parts) > 1 else "No reason"
    await message.answer(f"Banned {user_id}: {reason}")
```

### Text handlers with filters

```python
from aiogram import F, Router
from aiogram.types import Message

router = Router()


# Exact text match
@router.message(F.text == "ping")
async def pong(message: Message) -> None:
    await message.answer("pong")


# Starts with
@router.message(F.text.startswith("/custom_"))
async def custom_command(message: Message) -> None:
    await message.answer(f"Got: {message.text}")


# Contains (case-insensitive)
@router.message(F.text.lower().contains("help"))
async def help_mention(message: Message) -> None:
    await message.answer("Need help? Use /help")


# Regex
import re
@router.message(F.text.regexp(r"^#(\w+)"))
async def hashtag(message: Message) -> None:
    match = re.match(r"^#(\w+)", message.text)
    if match:
        await message.answer(f"Tag: {match.group(1)}")


# Any text (fallback -- register LAST)
@router.message(F.text)
async def echo(message: Message) -> None:
    await message.answer(message.text)
```

### Callback query handlers

```python
from aiogram import F, Router
from aiogram.types import CallbackQuery

router = Router()


@router.callback_query(F.data == "confirm")
async def on_confirm(callback: CallbackQuery) -> None:
    await callback.message.edit_text("Confirmed!")
    await callback.answer()  # dismiss loading spinner


# Prefix pattern
@router.callback_query(F.data.startswith("page:"))
async def on_page(callback: CallbackQuery) -> None:
    page = int(callback.data.split(":")[1])
    await callback.message.edit_text(f"Page {page}")
    await callback.answer()


# With alert popup
@router.callback_query(F.data == "info")
async def on_info(callback: CallbackQuery) -> None:
    await callback.answer("This is an alert!", show_alert=True)
```

### Content type handlers

```python
from aiogram import F, Router
from aiogram.types import Message

router = Router()


@router.message(F.photo)
async def on_photo(message: Message) -> None:
    photo = message.photo[-1]  # largest resolution
    file = await message.bot.get_file(photo.file_id)
    await message.answer(f"Photo received: {photo.width}x{photo.height}")


@router.message(F.document)
async def on_document(message: Message) -> None:
    doc = message.document
    await message.answer(f"File: {doc.file_name} ({doc.file_size} bytes)")


@router.message(F.voice)
async def on_voice(message: Message) -> None:
    voice = message.voice
    file = await message.bot.get_file(voice.file_id)
    # Download: await message.bot.download_file(file.file_path, destination)
    await message.answer(f"Voice: {voice.duration}s")


@router.message(F.location)
async def on_location(message: Message) -> None:
    loc = message.location
    await message.answer(f"Location: {loc.latitude}, {loc.longitude}")
```

### Error handler

```python
from aiogram import Dispatcher
from aiogram.types import ErrorEvent

dp = Dispatcher()


@dp.errors()
async def global_error_handler(event: ErrorEvent) -> bool:
    """Catch all unhandled exceptions."""
    import logging
    logging.exception(
        "Update %s caused error: %s",
        event.update.update_id,
        event.exception,
    )
    # Return True to suppress the exception
    # Return False to propagate
    return True
```

---

## 4. FSM (Finite State Machine)

### StatesGroup definition

```python
from aiogram.fsm.state import State, StatesGroup


class RegistrationForm(StatesGroup):
    waiting_for_name = State()
    waiting_for_email = State()
    waiting_for_confirm = State()
```

### State transitions

```python
from aiogram import F, Router
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.types import CallbackQuery, Message

from states.forms import RegistrationForm

router = Router()


@router.message(Command("register"))
async def cmd_register(message: Message, state: FSMContext) -> None:
    await state.set_state(RegistrationForm.waiting_for_name)
    await message.answer("Enter your name:")


@router.message(RegistrationForm.waiting_for_name, F.text)
async def process_name(message: Message, state: FSMContext) -> None:
    await state.update_data(name=message.text)
    await state.set_state(RegistrationForm.waiting_for_email)
    await message.answer("Enter your email:")


@router.message(RegistrationForm.waiting_for_email, F.text)
async def process_email(message: Message, state: FSMContext) -> None:
    if "@" not in message.text:
        await message.answer("Invalid email. Try again:")
        return

    await state.update_data(email=message.text)
    data = await state.get_data()
    await state.set_state(RegistrationForm.waiting_for_confirm)
    await message.answer(
        f"Confirm:\nName: {data['name']}\nEmail: {data['email']}\n\n"
        f"Send /confirm or /cancel"
    )


@router.message(RegistrationForm.waiting_for_confirm, Command("confirm"))
async def process_confirm(message: Message, state: FSMContext) -> None:
    data = await state.get_data()
    # Save to DB here
    await state.clear()  # clears state AND data
    await message.answer(f"Registered: {data['name']} ({data['email']})")


@router.message(Command("cancel"))
async def cmd_cancel(message: Message, state: FSMContext) -> None:
    current = await state.get_state()
    if current is None:
        await message.answer("Nothing to cancel.")
        return
    await state.clear()
    await message.answer("Cancelled.")
```

### Storage options

```python
from aiogram import Dispatcher

# --- Memory (dev only, lost on restart) ---
from aiogram.fsm.storage.memory import MemoryStorage
dp = Dispatcher(storage=MemoryStorage())

# --- Redis (production) ---
from aiogram.fsm.storage.redis import RedisStorage
storage = RedisStorage.from_url("redis://localhost:6379/0")
dp = Dispatcher(storage=storage)

# Redis with password
storage = RedisStorage.from_url("redis://:password@redis-host:6379/1")

# Redis with custom prefix and TTL
storage = RedisStorage.from_url(
    "redis://localhost:6379/0",
    state_ttl=3600,    # state expires in 1 hour
    data_ttl=3600,     # data expires in 1 hour
)
```

### FSM strategy (scoping)

```python
from aiogram import Dispatcher
from aiogram.fsm.strategy import FSMStrategy

# Default: one FSM state per user per chat
dp = Dispatcher(fsm_strategy=FSMStrategy.USER_IN_CHAT)

# Global per user (same state in all chats)
dp = Dispatcher(fsm_strategy=FSMStrategy.GLOBAL_USER)

# Per chat (all users share state -- rare, group games)
dp = Dispatcher(fsm_strategy=FSMStrategy.CHAT)
```

---

## 5. Middleware

### BaseMiddleware pattern (throttling)

```python
import time
from collections import defaultdict
from typing import Any, Awaitable, Callable

from aiogram import BaseMiddleware
from aiogram.types import TelegramObject, Message

RATE_LIMIT_SECONDS = 1


class ThrottlingMiddleware(BaseMiddleware):
    def __init__(self) -> None:
        self._last_call: dict[int, float] = defaultdict(float)

    async def __call__(
        self,
        handler: Callable[[TelegramObject, dict[str, Any]], Awaitable[Any]],
        event: TelegramObject,
        data: dict[str, Any],
    ) -> Any:
        if not isinstance(event, Message) or not event.from_user:
            return await handler(event, data)

        user_id = event.from_user.id
        now = time.monotonic()

        if now - self._last_call[user_id] < RATE_LIMIT_SECONDS:
            return None  # silently drop

        self._last_call[user_id] = now
        return await handler(event, data)
```

### Auth/admin check middleware

```python
from typing import Any, Awaitable, Callable

from aiogram import BaseMiddleware
from aiogram.types import Message, TelegramObject

ADMIN_IDS: set[int] = {164795011}


class AdminMiddleware(BaseMiddleware):
    async def __call__(
        self,
        handler: Callable[[TelegramObject, dict[str, Any]], Awaitable[Any]],
        event: TelegramObject,
        data: dict[str, Any],
    ) -> Any:
        if isinstance(event, Message) and event.from_user:
            data["is_admin"] = event.from_user.id in ADMIN_IDS
        return await handler(event, data)
```

Handler receives `is_admin` via kwargs:

```python
@router.message(Command("stats"))
async def cmd_stats(message: Message, is_admin: bool = False) -> None:
    if not is_admin:
        await message.answer("Access denied.")
        return
    await message.answer("Stats: ...")
```

### Database session middleware

```python
from typing import Any, Awaitable, Callable

from aiogram import BaseMiddleware
from aiogram.types import TelegramObject
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker


class DbSessionMiddleware(BaseMiddleware):
    def __init__(self, session_pool: async_sessionmaker[AsyncSession]) -> None:
        self._session_pool = session_pool

    async def __call__(
        self,
        handler: Callable[[TelegramObject, dict[str, Any]], Awaitable[Any]],
        event: TelegramObject,
        data: dict[str, Any],
    ) -> Any:
        async with self._session_pool() as session:
            data["db"] = session
            result = await handler(event, data)
            await session.commit()
            return result
```

Handler:

```python
@router.message(Command("profile"))
async def cmd_profile(message: Message, db: AsyncSession) -> None:
    user = await db.get(User, message.from_user.id)
    await message.answer(f"Profile: {user}")
```

### Registration

```python
# Register on Dispatcher (all update types)
dp.message.middleware(ThrottlingMiddleware())
dp.callback_query.middleware(ThrottlingMiddleware())

# Register on specific Router
admin_router.message.middleware(AdminMiddleware())

# Outer middleware (runs before filters, even if no handler matches)
dp.message.outer_middleware(DbSessionMiddleware(session_pool))
```

---

## 6. Keyboards

### InlineKeyboardMarkup builder

```python
from aiogram.types import InlineKeyboardButton, InlineKeyboardMarkup
from aiogram.utils.keyboard import InlineKeyboardBuilder


def main_menu_kb() -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    builder.row(
        InlineKeyboardButton(text="Profile", callback_data="menu:profile"),
        InlineKeyboardButton(text="Settings", callback_data="menu:settings"),
    )
    builder.row(
        InlineKeyboardButton(text="Help", callback_data="menu:help"),
    )
    builder.row(
        InlineKeyboardButton(
            text="Visit Website",
            url="https://example.com",
        ),
    )
    return builder.as_markup()
```

### ReplyKeyboardMarkup builder

```python
from aiogram.types import KeyboardButton, ReplyKeyboardMarkup
from aiogram.utils.keyboard import ReplyKeyboardBuilder


def contact_kb() -> ReplyKeyboardMarkup:
    builder = ReplyKeyboardBuilder()
    builder.row(
        KeyboardButton(text="Share Phone", request_contact=True),
        KeyboardButton(text="Share Location", request_location=True),
    )
    builder.row(KeyboardButton(text="Cancel"))
    return builder.as_markup(
        resize_keyboard=True,
        one_time_keyboard=True,
        input_field_placeholder="Choose an option",
    )
```

### Callback data with prefix pattern

```python
from aiogram import F, Router
from aiogram.filters.callback_data import CallbackData
from aiogram.types import CallbackQuery
from aiogram.utils.keyboard import InlineKeyboardBuilder


class ItemCallback(CallbackData, prefix="item"):
    action: str      # "view" | "buy" | "delete"
    item_id: int
    quantity: int = 1


def items_kb(items: list[dict[str, int | str]]) -> InlineKeyboardBuilder:
    builder = InlineKeyboardBuilder()
    for item in items:
        builder.button(
            text=str(item["name"]),
            callback_data=ItemCallback(
                action="view",
                item_id=int(item["id"]),
            ),
        )
    builder.adjust(2)  # 2 buttons per row
    return builder


router = Router()


@router.callback_query(ItemCallback.filter(F.action == "view"))
async def on_item_view(
    callback: CallbackQuery,
    callback_data: ItemCallback,
) -> None:
    await callback.message.edit_text(f"Item #{callback_data.item_id}")
    await callback.answer()


@router.callback_query(ItemCallback.filter(F.action == "buy"))
async def on_item_buy(
    callback: CallbackQuery,
    callback_data: ItemCallback,
) -> None:
    await callback.answer(
        f"Bought {callback_data.quantity}x item #{callback_data.item_id}",
        show_alert=True,
    )
```

### Pagination keyboard helper

```python
from aiogram.types import InlineKeyboardMarkup
from aiogram.utils.keyboard import InlineKeyboardBuilder


def pagination_kb(
    current_page: int,
    total_pages: int,
    prefix: str = "page",
) -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()

    buttons = []
    if current_page > 1:
        buttons.append(("< Prev", f"{prefix}:{current_page - 1}"))

    buttons.append((f"{current_page}/{total_pages}", f"{prefix}:noop"))

    if current_page < total_pages:
        buttons.append(("Next >", f"{prefix}:{current_page + 1}"))

    for text, data in buttons:
        builder.button(text=text, callback_data=data)

    builder.adjust(len(buttons))
    return builder.as_markup()
```

---

## 7. Magic Filters

### Common patterns

```python
from aiogram import F

# Text
F.text                          # message has text
F.text == "exact"               # exact match
F.text.lower() == "hello"       # case-insensitive
F.text.startswith("/")          # starts with
F.text.contains("keyword")     # contains
F.text.regexp(r"\d{4,6}")      # regex

# Callback
F.data == "confirm"             # exact callback_data
F.data.startswith("menu:")      # prefix
F.data.in_({"yes", "no"})      # one of set

# User
F.from_user.id == 164795011    # specific user
F.from_user.id.in_({1, 2, 3}) # user whitelist

# Chat type
F.chat.type == "private"
F.chat.type.in_({"group", "supergroup"})

# Content types
F.photo                         # has photo
F.document                      # has document
F.voice                         # has voice
F.sticker                       # has sticker
F.animation                     # has GIF

# Reply
F.reply_to_message              # is a reply
F.reply_to_message.from_user.id == F.bot.id  # reply to bot
```

### Combining filters

```python
from aiogram import F

# AND -- both conditions
F.text & F.chat.type == "private"

# OR -- either condition
(F.text == "yes") | (F.text == "da")

# NOT -- negate
~F.from_user.is_bot

# Complex: private chat AND (text or photo) AND not bot
(F.chat.type == "private") & (F.text | F.photo) & ~F.from_user.is_bot
```

### Custom filter class

```python
from aiogram.filters import Filter
from aiogram.types import Message


class IsAdminFilter(Filter):
    def __init__(self, admin_ids: list[int]) -> None:
        self._admin_ids = set(admin_ids)

    async def __call__(self, message: Message) -> bool | dict[str, bool]:
        if message.from_user and message.from_user.id in self._admin_ids:
            return {"is_admin": True}  # inject into handler kwargs
        return False  # filter rejects -- handler not called


# Usage
@router.message(Command("secret"), IsAdminFilter(admin_ids=[164795011]))
async def secret_cmd(message: Message, is_admin: bool) -> None:
    await message.answer("Admin area")
```

---

## 8. Dependency Injection

aiogram injects handler kwargs from the middleware `data` dict.

```python
# middleware -- inject service
class AiServiceMiddleware(BaseMiddleware):
    def __init__(self, ai_client: "OpenAIClient") -> None:
        self._ai = ai_client

    async def __call__(
        self,
        handler: Callable[[TelegramObject, dict[str, Any]], Awaitable[Any]],
        event: TelegramObject,
        data: dict[str, Any],
    ) -> Any:
        data["ai"] = self._ai
        return await handler(event, data)


# handler -- receive service
@router.message(F.text)
async def ai_reply(message: Message, ai: "OpenAIClient") -> None:
    response = await ai.complete(message.text)
    await message.answer(response)
```

Alternative: pass via `dp["key"]` (workflow data, available in all handlers):

```python
dp = Dispatcher()
dp["config"] = settings
dp["http_client"] = aiohttp.ClientSession()

# handler
@router.message(Command("status"))
async def cmd_status(message: Message, config: Settings) -> None:
    await message.answer(f"Debug: {config.debug}")
```

---

## 9. Entry Point

### Complete bot.py with polling

```python
import asyncio
import logging

from aiogram import Bot, Dispatcher
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.fsm.storage.redis import RedisStorage

from config import settings
from handlers import setup_routers
from middlewares.throttling import ThrottlingMiddleware

logging.basicConfig(
    level=logging.DEBUG if settings.debug else logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)
log = logging.getLogger(__name__)


async def main() -> None:
    storage = RedisStorage.from_url(
        settings.redis_url,
        state_ttl=3600,
        data_ttl=3600,
    )

    bot = Bot(
        token=settings.bot_token,
        default=DefaultBotProperties(parse_mode=ParseMode.HTML),
    )
    dp = Dispatcher(storage=storage)
    dp["config"] = settings

    dp.message.middleware(ThrottlingMiddleware())
    dp.include_router(setup_routers())

    log.info("Starting bot in polling mode")
    try:
        await dp.start_polling(
            bot,
            allowed_updates=dp.resolve_used_update_types(),
        )
    finally:
        await bot.session.close()


if __name__ == "__main__":
    asyncio.run(main())
```

### Complete bot.py with webhook (FastAPI + uvicorn)

```python
import logging
from contextlib import asynccontextmanager
from typing import AsyncGenerator

from aiogram import Bot, Dispatcher
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.fsm.storage.redis import RedisStorage
from aiogram.types import Update
from fastapi import FastAPI, Header, HTTPException, Request

from config import settings
from handlers import setup_routers
from middlewares.throttling import ThrottlingMiddleware

logging.basicConfig(
    level=logging.DEBUG if settings.debug else logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)
log = logging.getLogger(__name__)

storage = RedisStorage.from_url(settings.redis_url)

bot = Bot(
    token=settings.bot_token,
    default=DefaultBotProperties(parse_mode=ParseMode.HTML),
)
dp = Dispatcher(storage=storage)
dp["config"] = settings
dp.message.middleware(ThrottlingMiddleware())
dp.include_router(setup_routers())

WEBHOOK_PATH = "/webhook/telegram"


@asynccontextmanager
async def lifespan(_app: FastAPI) -> AsyncGenerator[None, None]:
    webhook_url = f"{settings.webhook_base_url}{WEBHOOK_PATH}"
    await bot.set_webhook(
        url=webhook_url,
        secret_token=settings.webhook_secret,
        allowed_updates=dp.resolve_used_update_types(),
    )
    log.info("Webhook set: %s", webhook_url)
    yield
    await bot.delete_webhook()
    await bot.session.close()
    await storage.close()
    log.info("Shutdown complete")


app = FastAPI(lifespan=lifespan)


@app.post(WEBHOOK_PATH)
async def webhook_handler(
    request: Request,
    x_telegram_bot_api_secret_token: str = Header(None),
) -> dict[str, bool]:
    if x_telegram_bot_api_secret_token != settings.webhook_secret:
        raise HTTPException(status_code=403)

    update = Update.model_validate(await request.json(), context={"bot": bot})
    await dp.feed_update(bot, update)
    return {"ok": True}


@app.get("/health")
async def health() -> dict[str, str]:
    return {"status": "ok"}


if __name__ == "__main__":
    import uvicorn

    uvicorn.run(app, host="0.0.0.0", port=8080)
```

### Graceful shutdown (polling)

```python
import asyncio
import signal
from types import FrameType


async def main() -> None:
    bot = Bot(token=settings.bot_token)
    dp = Dispatcher()

    loop = asyncio.get_running_loop()

    def shutdown(sig: int, frame: FrameType | None = None) -> None:
        log.info("Received signal %s, shutting down", sig)
        # stop_polling is cooperative -- finishes current update
        loop.call_soon_threadsafe(dp.stop_polling)

    signal.signal(signal.SIGINT, shutdown)
    signal.signal(signal.SIGTERM, shutdown)

    try:
        await dp.start_polling(bot)
    finally:
        await bot.session.close()
        log.info("Bot stopped")
```

---

## Quick Reference: pip install

```
aiogram>=3.28
pydantic-settings>=2.0
redis>=5.0                 # for RedisStorage
fastapi>=0.115             # for webhook mode
uvicorn[standard]>=0.30    # for webhook mode
aiohttp>=3.9               # already an aiogram dependency
```
