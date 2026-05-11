# Payments -- Telegram Bot Builder

Patterns for monetization: Telegram Stars, external providers, subscriptions.

## Table of Contents

1. Telegram Stars (Native Payments)
2. Star Subscriptions (Recurring)
3. External Providers (Stripe, CloudPayments, YooKassa)
4. Subscription Management Pattern
5. Deep Links for Payments
6. Payment Testing Checklist

## 1. Telegram Stars (Native Payments)

Stars are Telegram's built-in currency. No provider_token needed.
1 Star ~ $0.02 USD (approximate, varies).

### Send invoice

```python
from aiogram import Router, F
from aiogram.types import (
    Message,
    LabeledPrice,
    PreCheckoutQuery,
    ContentType,
)

router = Router()

PRICES = {
    "basic": {"stars": 100, "label": "Basic Plan"},
    "pro": {"stars": 500, "label": "Pro Plan"},
}


@router.message(F.text == "/buy")
async def send_star_invoice(message: Message) -> None:
    """Send a Telegram Stars payment invoice."""
    plan = PRICES["basic"]
    await message.answer_invoice(
        title=plan["label"],
        description="Access to premium features for 30 days",
        payload=f"sub_basic_{message.from_user.id}",
        currency="XTR",  # XTR = Telegram Stars
        prices=[LabeledPrice(label=plan["label"], amount=plan["stars"])],
        provider_token="",  # empty for Stars
    )
```

### Pre-checkout query handler (must respond in <10 seconds)

```python
@router.pre_checkout_query()
async def handle_pre_checkout(query: PreCheckoutQuery) -> None:
    """Validate payment before charging.

    MUST respond within 10 seconds or payment fails.
    """
    payload = query.invoice_payload

    # validate payload format
    if not payload.startswith("sub_"):
        await query.answer(ok=False, error_message="Invalid order")
        return

    # check stock / eligibility here (fast check only)
    await query.answer(ok=True)
```

### Successful payment handler

```python
@router.message(F.content_type == ContentType.SUCCESSFUL_PAYMENT)
async def handle_payment_success(message: Message) -> None:
    """Process successful Stars payment."""
    payment = message.successful_payment
    user_id = message.from_user.id

    # payment.telegram_payment_charge_id -- unique Telegram charge ID
    # payment.total_amount -- Stars paid
    # payment.invoice_payload -- your custom payload

    # activate subscription in DB
    await activate_subscription(
        user_id=user_id,
        plan=payment.invoice_payload,
        charge_id=payment.telegram_payment_charge_id,
        amount=payment.total_amount,
    )

    await message.answer(
        "Payment successful! Premium activated for 30 days."
    )
```

### Refund Stars

```python
async def refund_stars(
    bot, user_id: int, charge_id: str
) -> bool:
    """Refund a Stars payment."""
    try:
        await bot.refund_star_payment(
            user_id=user_id,
            telegram_payment_charge_id=charge_id,
        )
        return True
    except Exception:
        return False
```

## 2. Star Subscriptions (Recurring)

Telegram supports native recurring Star subscriptions.

### Create subscription invoice link

```python
async def create_subscription_link(bot) -> str:
    """Create a subscription invoice link (monthly renewal)."""
    link = await bot.create_invoice_link(
        title="Monthly Premium",
        description="Premium features, renews monthly",
        payload="monthly_premium",
        currency="XTR",
        prices=[LabeledPrice(label="Monthly Premium", amount=250)],
        provider_token="",
        subscription_period=2592000,  # 30 days in seconds
    )
    return link
```

### Handle subscription renewal

Telegram automatically charges and sends `successful_payment` updates
on renewal. The same handler (section 1) processes renewals.
Distinguish initial vs renewal by checking your DB for existing
subscription records.

## 3. External Providers (Stripe, CloudPayments, YooKassa)

Same invoice pattern but with a provider_token from @BotFather.

### Setup

1. `/mybots` -> select bot -> Payments -> choose provider
2. Get `provider_token` (test and live)
3. Store in env: `PAYMENT_PROVIDER_TOKEN`

### Stripe example

```python
import os

PROVIDER_TOKEN = os.getenv("PAYMENT_PROVIDER_TOKEN")


@router.message(F.text == "/subscribe")
async def send_stripe_invoice(message: Message) -> None:
    """Send invoice via external payment provider."""
    await message.answer_invoice(
        title="Pro Plan",
        description="Premium access for 30 days",
        payload=f"stripe_pro_{message.from_user.id}",
        currency="USD",  # real currency
        prices=[
            LabeledPrice(label="Pro Plan", amount=999),  # $9.99
        ],
        provider_token=PROVIDER_TOKEN,
        need_email=True,  # collect email for receipt
    )
```

### CloudPayments (RUB)

```python
@router.message(F.text == "/pay_rub")
async def send_cloudpayments_invoice(message: Message) -> None:
    """Send RUB invoice via CloudPayments."""
    await message.answer_invoice(
        title="Premium",
        description="Premium on 30 days",
        payload=f"cp_premium_{message.from_user.id}",
        currency="RUB",
        prices=[
            LabeledPrice(label="Premium", amount=99900),  # 999.00 RUB
        ],
        provider_token=PROVIDER_TOKEN,
        need_email=True,
    )
```

Pre-checkout and successful_payment handlers are identical to Stars
(section 1). Only `currency` and `provider_token` differ.

### YooKassa (Russian market)

Same pattern as CloudPayments. Get provider_token from @BotFather ->
YooKassa. Currency: RUB. Amount in kopecks (100 = 1 RUB).

## 4. Subscription Management Pattern

### Middleware to check subscription status

```python
from aiogram import BaseMiddleware
from aiogram.types import TelegramObject, Message


class SubscriptionMiddleware(BaseMiddleware):
    """Block premium commands for non-subscribers."""

    def __init__(self, premium_commands: set[str]) -> None:
        self.premium_commands = premium_commands

    async def __call__(
        self,
        handler,
        event: TelegramObject,
        data: dict,
    ):
        if not isinstance(event, Message):
            return await handler(event, data)

        if not event.text:
            return await handler(event, data)

        command = event.text.split()[0].lstrip("/")
        if command not in self.premium_commands:
            return await handler(event, data)

        # check subscription in DB
        user_id = event.from_user.id
        is_active = await check_subscription_active(user_id)

        if not is_active:
            await event.reply(
                "This feature requires a subscription.\n"
                "Use /buy to subscribe."
            )
            return

        data["subscription"] = await get_subscription(user_id)
        return await handler(event, data)
```

### Freemium: free tier with daily limits

```python
import redis.asyncio as redis

FREE_DAILY_LIMIT = 5
PAID_DAILY_LIMIT = 0  # unlimited

async def check_tier_limit(
    user_id: int, is_paid: bool
) -> tuple[bool, int]:
    """Check freemium usage limit.

    Returns:
        (allowed, remaining)
    """
    if is_paid:
        return True, -1  # unlimited

    r = redis.Redis(connection_pool=pool)
    key = f"tier:{user_id}:daily"
    count = await r.incr(key)
    if count == 1:
        await r.expire(key, 86400)

    remaining = max(0, FREE_DAILY_LIMIT - count)
    return count <= FREE_DAILY_LIMIT, remaining
```

### Soft degradation vs hard block

```python
async def handle_with_degradation(
    message: Message, is_paid: bool
) -> None:
    """Paid users get GPT-4o, free users get GPT-4o-mini."""
    model = "gpt-4o" if is_paid else "gpt-4o-mini"
    max_tokens = 4096 if is_paid else 1024

    # use cheaper model for free tier instead of blocking
    response = await client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": message.text}],
        max_tokens=max_tokens,
    )
    await message.reply(response.choices[0].message.content)
```

## 5. Deep Links for Payments

### t.me/bot?start=pay_PLAN pattern

```python
@router.message(F.text.startswith("/start"))
async def handle_start(message: Message) -> None:
    """Handle /start with optional deep link payload."""
    args = message.text.split(maxsplit=1)
    payload = args[1] if len(args) > 1 else ""

    # deep link: t.me/bot?start=pay_pro
    if payload.startswith("pay_"):
        plan_id = payload.removeprefix("pay_")
        if plan_id in PRICES:
            await send_invoice_for_plan(message, plan_id)
            return
        await message.reply("Unknown plan.")
        return

    # deep link: t.me/bot?start=ref_USER_ID
    if payload.startswith("ref_"):
        referrer_id = payload.removeprefix("ref_")
        await save_referral(message.from_user.id, referrer_id)

    # normal /start
    await message.reply(
        "Welcome! Use /buy to see plans."
    )


async def send_invoice_for_plan(
    message: Message, plan_id: str
) -> None:
    """Send invoice for a specific plan."""
    plan = PRICES[plan_id]
    await message.answer_invoice(
        title=plan["label"],
        description=f"Subscribe to {plan['label']}",
        payload=f"sub_{plan_id}_{message.from_user.id}",
        currency="XTR",
        prices=[
            LabeledPrice(
                label=plan["label"], amount=plan["stars"]
            )
        ],
        provider_token="",
    )
```

### Generate payment deep links dynamically

```python
BOT_USERNAME = "MyBot"

def payment_link(plan_id: str) -> str:
    """Generate a payment deep link."""
    return f"https://t.me/{BOT_USERNAME}?start=pay_{plan_id}"

# use in inline keyboards:
# InlineKeyboardButton(text="Buy Pro", url=payment_link("pro"))
```

## Payment Testing Checklist

- [ ] Test mode provider_token for external providers
- [ ] Stars: use small amounts (1-5) for testing
- [ ] Pre-checkout handler responds within 10 seconds
- [ ] Successful payment handler is idempotent (duplicate charges safe)
- [ ] Payload contains enough info to identify user + plan
- [ ] Refund flow tested
- [ ] DB transaction: charge_id stored before confirming to user
- [ ] Error in post-payment logic does not lose the payment record
