# Layer 3: Telegram bridge

Два репо, один из них canonical, другой – deprecated.

- **Canonical:** [`qwwiwi/dashi-plugin-claude-code`](https://github.com/qwwiwi/dashi-plugin-claude-code)
- **Deprecated:** [`qwwiwi/jarvis-telegram-gateway`](https://github.com/qwwiwi/jarvis-telegram-gateway)

## Зачем мост

Claude Code запускается в терминале. Если вы не сидите за компьютером – агент для вас недоступен. Telegram-мост решает это: вы пишете боту, бот пробрасывает сообщение в агента, агент отвечает обратно в Telegram. Получаете полноценного AI-помощника в кармане.

## Plugin pattern vs Gateway pattern

Это два разных способа сделать одно и то же. Различия принципиальные.

| Аспект | Plugin (canonical) | Gateway (deprecated) |
|---|---|---|
| Как запускается | Внутри сессии Claude Code как channel-plugin | Отдельный процесс, вызывает `claude -p` под капотом |
| Биллинг | Идёт по вашей Max-подписке | После 2026-06-15 идёт по отдельному SDK-пулу ($200/mo) |
| Скорость отклика | Быстро – нет cold start на каждое сообщение | Медленнее – cold start на каждый `claude -p` запуск |
| Состояние сессии | Сохраняется между сообщениями | Каждый запрос – новая сессия |
| Установка | Один процесс на агента, Bun + TS | Один процесс на агента, Python |

## Почему gateway deprecated

Anthropic объявили: с 15 июня 2026 биллинг `claude -p` (non-interactive mode) выделяется в отдельный SDK-пул, который НЕ покрывается Max-подпиской. Это означает, что для каждого Telegram-сообщения через gateway вы будете платить отдельно по тарифу SDK.

Plugin-pattern работает иначе: он живёт внутри обычной интерактивной сессии Claude Code и расходует ту же Max-подписку, что и работа в терминале. После 15 июня plugin продолжит работать в рамках одной подписки, gateway станет стоить отдельных $200/mo за каждого активного агента.

**Дедлайн:**
- До 2026-06-15 – оба варианта работают идентично по стоимости
- После 2026-06-15 – gateway становится платным, plugin остаётся в рамках Max
- 2026-09-15 – архивация репо gateway, дальше только historical reference

## Что выбирать новым ученикам

**Всегда plugin.** Не тратьте время на установку gateway, даже если найдёте его упомянутым в старых гайдах. Plugin – единственный правильный путь для новых установок.

## Как устанавливается plugin

Один плагин на каждого агента. Внутри агента вы делаете:

```bash
cd ~/.claude-lab/{agent-name}/.claude/
git clone https://github.com/qwwiwi/dashi-plugin-claude-code.git plugins/dashi-channel
cd plugins/dashi-channel
bun install
cp config.example.json config.json
# заполняете токен Telegram-бота и user_id принимающего
bun run start
```

Plugin регистрируется в Claude Code как channel и начинает слушать Telegram-апдейты. Когда боту приходит сообщение от вашего user_id, plugin пробрасывает его как новое сообщение в текущую сессию агента. Ответ агента уходит обратно тому же пользователю.

## Что внутри plugin

- **Bun + TypeScript** – легковесный runtime, быстрый старт.
- **Long-polling Telegram API** – без webhook, не нужен публичный endpoint.
- **Channel API Claude Code** – стандартный механизм plugin, без хаков.
- **State persistence** – сохраняет историю в локальный файл, переживает рестарт.

## Один процесс – один агент

Plugin запускается внутри workspace конкретного агента. Для трёх агентов (coder, marketer, sales) – три отдельных plugin-процесса. Каждый со своим Telegram-ботом, своим конфигом, своим user_id.

Это даёт изоляцию: если plugin coder-а упал, marketer и sales продолжают работать.

## Schedule deprecation

| Дата | Что меняется |
|---|---|
| 2026-05-19 | Текущая ситуация: оба варианта работают, gateway маркирован deprecated |
| 2026-06-15 | Anthropic billing split: claude -p уходит в SDK pool |
| 2026-09-15 | Архивация репо gateway, новые установки только через plugin |

Если у вас уже есть рабочий gateway, мигрируйте до 2026-06-15.

## Migration path

В репо `dashi-plugin-claude-code` есть гайд по миграции с gateway:
`docs/04-migration-from-gateway.md`

Краткая суть:
1. Остановить gateway процесс.
2. Сохранить конфиг (токены, user_id, allowlist).
3. Установить plugin в workspace.
4. Перенести конфиг в `config.json` plugin.
5. Запустить, проверить smoke-test.
6. Удалить gateway директорию.

Миграция занимает ~30 минут на агента.

## Внешние ссылки

- Canonical (использовать): https://github.com/qwwiwi/dashi-plugin-claude-code
- Deprecated (только историческая справка): https://github.com/qwwiwi/jarvis-telegram-gateway
