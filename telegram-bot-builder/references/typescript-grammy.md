---
last_verified: 2026-05-11
applies_to: [grammy, typescript, telegram-bot]
confidence: high
grammy_version: "1.35"
---

# grammY TypeScript Reference

Copy-paste-ready patterns for building Telegram bots with grammY.

---

## Table of Contents

1. Bot Setup
2. Middleware
3. Handlers
4. Conversations Plugin
5. Sessions
6. Keyboards
7. Plugins
8. Serverless Deployment
9. Entry Point

## 1. Bot Setup

### Basic bot with polling

```typescript
import { Bot } from "grammy";

const bot = new Bot(process.env.BOT_TOKEN!);

bot.command("start", (ctx) => ctx.reply("Hello!"));
bot.on("message:text", (ctx) => ctx.reply(`Echo: ${ctx.message.text}`));

bot.start();
```

### Bot with webhook (Express)

```typescript
import express from "express";
import { Bot, webhookCallback } from "grammy";

const bot = new Bot(process.env.BOT_TOKEN!);

bot.command("start", (ctx) => ctx.reply("Hello!"));

const app = express();
app.use(express.json());
app.post("/webhook", webhookCallback(bot, "express"));

app.listen(3000, async () => {
  await bot.api.setWebhook("https://example.com/webhook");
  console.log("Webhook set");
});
```

### Bot with webhook (Cloudflare Workers)

```typescript
import { Bot, webhookCallback } from "grammy";

export default {
  async fetch(req: Request, env: Record<string, string>): Promise<Response> {
    const bot = new Bot(env.BOT_TOKEN);
    bot.command("start", (ctx) => ctx.reply("Hello from Workers!"));
    return webhookCallback(bot, "cloudflare-mod")(req);
  },
};
```

### Config from env

```typescript
interface BotConfig {
  token: string;
  webhookUrl: string;
  adminId: number;
  redisUrl: string;
}

function loadConfig(): BotConfig {
  const token = process.env.BOT_TOKEN;
  if (!token) throw new Error("BOT_TOKEN is required");

  return {
    token,
    webhookUrl: process.env.WEBHOOK_URL ?? "",
    adminId: Number(process.env.ADMIN_ID ?? "0"),
    redisUrl: process.env.REDIS_URL ?? "redis://localhost:6379",
  };
}

const config = loadConfig();
const bot = new Bot(config.token);
```

## 2. Middleware

### Koa-style middleware pattern

Every middleware MUST call `await next()` to pass control to the next handler.
Forgetting `next()` silently swallows updates.

```typescript
import { Bot, Context, NextFunction } from "grammy";

// Logging middleware
bot.use(async (ctx: Context, next: NextFunction) => {
  const start = Date.now();
  await next(); // MUST await -- handlers after this run first
  const ms = Date.now() - start;
  console.log(`${ctx.update.update_id} processed in ${ms}ms`);
});
```

### Session middleware with storage adapters

```typescript
import { Bot, Context, session, SessionFlavor } from "grammy";

interface SessionData {
  step: string;
  count: number;
}

type BotContext = Context & SessionFlavor<SessionData>;

const bot = new Bot<BotContext>(process.env.BOT_TOKEN!);

bot.use(
  session({
    initial: (): SessionData => ({ step: "idle", count: 0 }),
  })
);

bot.command("count", async (ctx) => {
  ctx.session.count++;
  await ctx.reply(`Count: ${ctx.session.count}`);
});
```

### Auth middleware

```typescript
const ADMIN_IDS = new Set([164795011]);

async function adminOnly(ctx: Context, next: NextFunction): Promise<void> {
  if (!ctx.from || !ADMIN_IDS.has(ctx.from.id)) {
    await ctx.reply("Access denied.");
    return; // do NOT call next()
  }
  await next();
}

bot.command("admin", adminOnly, (ctx) => ctx.reply("Admin panel"));
```

### Error handling (bot.catch)

```typescript
bot.catch((err) => {
  const ctx = err.ctx;
  const e = err.error;

  if (e instanceof GrammyError) {
    console.error(`Grammy error in ${ctx.update.update_id}:`, e.description);
  } else if (e instanceof HttpError) {
    console.error("Network error:", e);
  } else {
    console.error("Unknown error:", e);
  }
});
```

## 3. Handlers

### Command handlers

```typescript
bot.command("start", (ctx) => ctx.reply("Welcome!"));
bot.command("help", (ctx) =>
  ctx.reply("Commands:\n/start -- begin\n/help -- this message")
);

// Command with arguments: /ban 123456
bot.command("ban", async (ctx) => {
  const userId = Number(ctx.match);
  if (!userId) return ctx.reply("Usage: /ban <user_id>");
  await ctx.reply(`Banned ${userId}`);
});
```

### Text handlers

```typescript
// Exact text match via hears (string or regex)
bot.hears("ping", (ctx) => ctx.reply("pong"));
bot.hears(/^\/echo (.+)$/, (ctx) => ctx.reply(ctx.match[1]));

// All text messages
bot.on("message:text", (ctx) => {
  console.log(`${ctx.from.id}: ${ctx.message.text}`);
});

// Photos, documents, etc.
bot.on("message:photo", (ctx) => ctx.reply("Nice photo!"));
bot.on("message:document", (ctx) => ctx.reply("Got document"));
```

### Callback query handlers

```typescript
bot.callbackQuery("confirm", async (ctx) => {
  await ctx.answerCallbackQuery({ text: "Confirmed!" });
  await ctx.editMessageText("Done.");
});

// Pattern match with regex
bot.callbackQuery(/^vote:(\w+)$/, async (ctx) => {
  const option = ctx.match[1];
  await ctx.answerCallbackQuery({ text: `Voted for ${option}` });
});
```

### Inline queries

```typescript
import { InlineQueryResultArticle } from "grammy/types";

bot.on("inline_query", async (ctx) => {
  const query = ctx.inlineQuery.query;

  const results: InlineQueryResultArticle[] = [
    {
      type: "article",
      id: "1",
      title: `Result for "${query}"`,
      input_message_content: { message_text: `You searched: ${query}` },
    },
  ];

  await ctx.answerInlineQuery(results, { cache_time: 30 });
});
```

### Composer pattern for grouping handlers

```typescript
import { Composer } from "grammy";

// Group related handlers into a module
const adminModule = new Composer<BotContext>();
adminModule.use(adminOnly); // middleware applied to all handlers below
adminModule.command("stats", (ctx) => ctx.reply("Stats: ..."));
adminModule.command("broadcast", (ctx) => ctx.reply("Broadcast: ..."));

const userModule = new Composer<BotContext>();
userModule.command("profile", (ctx) => ctx.reply("Your profile"));
userModule.command("settings", (ctx) => ctx.reply("Settings"));

// Register modules
bot.use(adminModule);
bot.use(userModule);
```

## 4. Conversations Plugin

### Setup and registration

```typescript
import {
  type Conversation,
  type ConversationFlavor,
  conversations,
  createConversation,
} from "@grammyjs/conversations";

type BotContext = Context & SessionFlavor<SessionData> & ConversationFlavor;
type BotConversation = Conversation<BotContext>;

const bot = new Bot<BotContext>(process.env.BOT_TOKEN!);

bot.use(session({ initial: (): SessionData => ({ step: "idle", count: 0 }) }));
bot.use(conversations());
bot.use(createConversation(onboardingConversation));

bot.command("onboard", async (ctx) => {
  await ctx.conversation.enter("onboardingConversation");
});
```

### Basic conversation (waitFor)

```typescript
async function onboardingConversation(
  conversation: BotConversation,
  ctx: BotContext
): Promise<void> {
  await ctx.reply("What is your name?");
  const nameCtx = await conversation.waitFor("message:text");
  const name = nameCtx.message.text;

  await ctx.reply(`Hello, ${name}! Send me your photo.`);
  const photoCtx = await conversation.waitFor("message:photo");

  await ctx.reply(`Thanks, ${name}! Onboarding complete.`);
}
```

### Multi-step form

```typescript
async function registrationForm(
  conversation: BotConversation,
  ctx: BotContext
): Promise<void> {
  await ctx.reply("Enter your email:");
  const emailCtx = await conversation.waitFor("message:text");
  const email = emailCtx.message.text;

  if (!email.includes("@")) {
    await ctx.reply("Invalid email. Try again.");
    return; // exits conversation, user must /register again
  }

  await ctx.reply("Choose your plan:", {
    reply_markup: new InlineKeyboard()
      .text("Free", "plan:free")
      .text("Pro", "plan:pro"),
  });

  const planCtx = await conversation.waitForCallbackQuery(/^plan:/);
  const plan = planCtx.match![0].replace("plan:", "");
  await planCtx.answerCallbackQuery();

  await ctx.reply(`Registered: ${email}, plan: ${plan}`);
}
```

### conversation.external() for side effects

Side effects (API calls, DB writes) MUST be wrapped in `conversation.external()`.
Conversations replay from the start on every update -- without external(),
side effects execute multiple times.

```typescript
async function paymentConversation(
  conversation: BotConversation,
  ctx: BotContext
): Promise<void> {
  await ctx.reply("Processing payment...");

  // DB call wrapped in external() -- runs only once
  const user = await conversation.external(async () => {
    return await db.users.findUnique({ where: { tgId: ctx.from!.id } });
  });

  if (!user) {
    await ctx.reply("User not found. /start first.");
    return;
  }

  const paymentUrl = await conversation.external(async () => {
    return await createPaymentLink(user.id, 2990);
  });

  await ctx.reply(`Pay here: ${paymentUrl}`);
}
```

## 5. Sessions

### In-memory sessions (default, lost on restart)

```typescript
bot.use(session({ initial: (): SessionData => ({ step: "idle", count: 0 }) }));
```

### Redis sessions

```typescript
import { RedisAdapter } from "@grammyjs/storage-redis";
import { createClient } from "redis";

const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();

bot.use(
  session({
    initial: (): SessionData => ({ step: "idle", count: 0 }),
    storage: new RedisAdapter({ instance: redis }),
  })
);
```

### Custom session data interface

```typescript
interface SessionData {
  step: "idle" | "awaiting_email" | "awaiting_payment";
  email: string | null;
  locale: "ru" | "en";
  cart: Array<{ productId: string; quantity: number }>;
}

type BotContext = Context & SessionFlavor<SessionData>;

const initial = (): SessionData => ({
  step: "idle",
  email: null,
  locale: "ru",
  cart: [],
});
```

### Session with getSessionKey (multi-chat)

```typescript
bot.use(
  session({
    initial,
    // Default key: `${ctx.chat?.id}:${ctx.from?.id}`
    // Override for per-user sessions across all chats:
    getSessionKey: (ctx) => ctx.from?.id.toString(),
  })
);
```

## 6. Keyboards

### InlineKeyboard builder

```typescript
import { InlineKeyboard } from "grammy";

const keyboard = new InlineKeyboard()
  .text("Option A", "opt:a")
  .text("Option B", "opt:b")
  .row()
  .url("Open Site", "https://example.com")
  .row()
  .switchInline("Share", "query");

await ctx.reply("Choose:", { reply_markup: keyboard });
```

### Keyboard (reply keyboard) builder

```typescript
import { Keyboard } from "grammy";

const keyboard = new Keyboard()
  .text("Profile").text("Settings").row()
  .text("Help")
  .resized()     // fit keyboard to buttons
  .oneTime();    // hide after tap

await ctx.reply("Menu:", { reply_markup: keyboard });
```

### Menu plugin

```typescript
import { Menu } from "@grammyjs/menu";

const mainMenu = new Menu<BotContext>("main-menu")
  .text("Profile", (ctx) => ctx.reply("Your profile"))
  .row()
  .submenu("Settings", "settings-menu", (ctx) =>
    ctx.editMessageText("Settings:")
  );

const settingsMenu = new Menu<BotContext>("settings-menu")
  .text("Language", (ctx) => ctx.reply("Language: RU"))
  .row()
  .back("Back", (ctx) => ctx.editMessageText("Main menu:"));

mainMenu.register(settingsMenu);
bot.use(mainMenu);

bot.command("menu", (ctx) =>
  ctx.reply("Main menu:", { reply_markup: mainMenu })
);
```

### Callback data parsing

```typescript
// Simple prefix pattern
bot.callbackQuery(/^action:(\w+):(\d+)$/, async (ctx) => {
  const [, action, id] = ctx.match!;
  // action = "approve", id = "42"
  await ctx.answerCallbackQuery({ text: `${action} ${id}` });
});

// Typed callback data with a helper
function encodeCallback(action: string, id: number): string {
  return `${action}:${id}`;
}

function decodeCallback(data: string): { action: string; id: number } {
  const [action, rawId] = data.split(":");
  return { action, id: Number(rawId) };
}
```

## 7. Plugins

### Auto-retry plugin

```typescript
import { autoRetry } from "@grammyjs/auto-retry";

bot.api.config.use(autoRetry({
  maxRetryAttempts: 3,
  maxDelaySeconds: 60,
}));
```

### Parse mode plugin

```typescript
import { hydrateReply, parseMode } from "@grammyjs/parse-mode";
import type { ParseModeFlavor } from "@grammyjs/parse-mode";

type BotContext = ParseModeFlavor<Context>;

const bot = new Bot<BotContext>(process.env.BOT_TOKEN!);
bot.use(hydrateReply);
bot.api.config.use(parseMode("HTML"));

// Now all replies default to HTML
bot.command("start", (ctx) => ctx.reply("<b>Bold</b> and <i>italic</i>"));

// Override per-call
bot.command("md", (ctx) =>
  ctx.reply("*Bold* and _italic_", { parse_mode: "MarkdownV2" })
);
```

### Hydrate plugin

Adds shortcut methods to API objects (edit, delete, pin, etc.).

```typescript
import { hydrate, HydrateFlavor } from "@grammyjs/hydrate";

type BotContext = HydrateFlavor<Context>;
const bot = new Bot<BotContext>(process.env.BOT_TOKEN!);
bot.use(hydrate());

bot.command("start", async (ctx) => {
  const msg = await ctx.reply("Loading...");
  // msg has shortcut methods:
  await msg.editText("Done!");
  // instead of: await ctx.api.editMessageText(ctx.chat.id, msg.message_id, "Done!")
});
```

### Rate limiter

```typescript
import { limit } from "@grammyjs/ratelimiter";

bot.use(
  limit({
    timeFrame: 2000,    // ms
    limit: 3,           // max updates per timeFrame per chat
    onLimitExceeded: (ctx) => ctx.reply("Slow down!"),
  })
);
```

## 8. Serverless Deployment

### Cloudflare Workers handler

```typescript
// src/index.ts
import { Bot, webhookCallback } from "grammy";

interface Env {
  BOT_TOKEN: string;
  BOT_INFO: string; // JSON from getMe(), avoids cold start call
}

export default {
  async fetch(req: Request, env: Env): Promise<Response> {
    const bot = new Bot(env.BOT_TOKEN, {
      botInfo: JSON.parse(env.BOT_INFO),
    });
    bot.command("start", (ctx) => ctx.reply("Hello from CF Workers!"));
    return webhookCallback(bot, "cloudflare-mod")(req);
  },
};
```

### Vercel Edge handler

```typescript
// api/webhook.ts
import { Bot, webhookCallback } from "grammy";

const bot = new Bot(process.env.BOT_TOKEN!);
bot.command("start", (ctx) => ctx.reply("Hello from Vercel!"));

export const config = { runtime: "edge" };

export default webhookCallback(bot, "std/http");
```

### Supabase Edge Function handler

```typescript
// supabase/functions/telegram-bot/index.ts
import { Bot, webhookCallback } from "grammy";

const bot = new Bot(Deno.env.get("BOT_TOKEN")!);
bot.command("start", (ctx) => ctx.reply("Hello from Supabase!"));

const handleUpdate = webhookCallback(bot, "std/http");

Deno.serve(async (req: Request) => {
  if (req.method === "POST") {
    return await handleUpdate(req);
  }
  return new Response("OK", { status: 200 });
});
```

### Setting BOT_INFO to skip getMe on cold start

Every `new Bot(token)` calls `getMe()` on first use. On serverless, this
adds 100-300ms to every cold start. Pass `botInfo` to skip it.

```bash
# Get bot info once:
curl "https://api.telegram.org/bot<TOKEN>/getMe" | jq '.result'
```

```typescript
// Store result as env var BOT_INFO (JSON string)
const bot = new Bot(process.env.BOT_TOKEN!, {
  botInfo: JSON.parse(process.env.BOT_INFO!),
});
```

## 9. Entry Point

### Complete bot.ts with polling

```typescript
import {
  Bot, Context, session, SessionFlavor, GrammyError, HttpError,
} from "grammy";
import { autoRetry } from "@grammyjs/auto-retry";
import { hydrateReply, parseMode } from "@grammyjs/parse-mode";
import type { ParseModeFlavor } from "@grammyjs/parse-mode";

// --- Types ---
interface SessionData {
  step: string;
}
type BotContext = ParseModeFlavor<Context> & SessionFlavor<SessionData>;

// --- Init ---
const token = process.env.BOT_TOKEN;
if (!token) throw new Error("BOT_TOKEN is required");

const bot = new Bot<BotContext>(token);

// --- Plugins ---
bot.api.config.use(autoRetry({ maxRetryAttempts: 3 }));
bot.use(hydrateReply);
bot.api.config.use(parseMode("HTML"));
bot.use(session({ initial: (): SessionData => ({ step: "idle" }) }));

// --- Handlers ---
bot.command("start", (ctx) => ctx.reply("<b>Welcome!</b>"));
bot.on("message:text", (ctx) => ctx.reply(`Echo: ${ctx.message.text}`));

// --- Error handler ---
bot.catch((err) => {
  const e = err.error;
  if (e instanceof GrammyError) {
    console.error("Grammy error:", e.description);
  } else if (e instanceof HttpError) {
    console.error("HTTP error:", e);
  } else {
    console.error("Error:", e);
  }
});

// --- Start ---
bot.start({
  onStart: (info) => console.log(`Bot @${info.username} started (polling)`),
});
```

### Complete bot.ts with Express webhook

```typescript
import express from "express";
import { Bot, Context, session, SessionFlavor, webhookCallback } from "grammy";
import { autoRetry } from "@grammyjs/auto-retry";

interface SessionData {
  step: string;
}
type BotContext = Context & SessionFlavor<SessionData>;

const token = process.env.BOT_TOKEN;
const webhookUrl = process.env.WEBHOOK_URL;
const port = Number(process.env.PORT ?? "3000");
const secretToken = process.env.WEBHOOK_SECRET ?? "";

if (!token) throw new Error("BOT_TOKEN is required");
if (!webhookUrl) throw new Error("WEBHOOK_URL is required");

const bot = new Bot<BotContext>(token);
bot.api.config.use(autoRetry({ maxRetryAttempts: 3 }));
bot.use(session({ initial: (): SessionData => ({ step: "idle" }) }));

bot.command("start", (ctx) => ctx.reply("Welcome!"));

bot.catch((err) => console.error("Bot error:", err.error));

const app = express();
app.use(express.json());

app.get("/health", (_req, res) => res.json({ status: "ok" }));
app.post(
  "/webhook",
  webhookCallback(bot, "express", { secretToken })
);

app.listen(port, async () => {
  await bot.api.setWebhook(webhookUrl, { secret_token: secretToken });
  console.log(`Webhook set to ${webhookUrl}, listening on :${port}`);
});
```

### TypeScript config (tsconfig.json essentials)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "resolveJsonModule": true,
    "sourceMap": true
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

### package.json essentials

```json
{
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/bot.ts",
    "build": "tsc",
    "start": "node dist/bot.js"
  },
  "dependencies": {
    "grammy": "^1.35.0",
    "@grammyjs/auto-retry": "^2.0.2",
    "@grammyjs/conversations": "^2.0.3",
    "@grammyjs/hydrate": "^2.0.1",
    "@grammyjs/menu": "^2.0.2",
    "@grammyjs/parse-mode": "^2.0.2",
    "@grammyjs/ratelimiter": "^2.0.1",
    "@grammyjs/storage-redis": "^2.5.0"
  },
  "devDependencies": {
    "typescript": "^5.7.0",
    "tsx": "^4.19.0",
    "@types/node": "^22.0.0"
  }
}
```
