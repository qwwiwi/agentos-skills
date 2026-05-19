# 09. Troubleshooting

Восемь типичных проблем, с которыми сталкивались ученики при прохождении урока. Решения проверены, можно копировать.

## 1. Bearer token 401 Unauthorized

**Симптом:** любой вызов через MCP возвращает 401, в логах Caddy `401 from agent_tokens lookup miss`.

**Причина:** sha256 hash в `agent_tokens` table не совпадает с тем, что считается от токена в `.mcp.json`.

**Чаще всего:** при сохранении токена в файл попал trailing `\n`.

**Проверка:**

```bash
# Считаем правильно (без \n)
TOKEN=$(cat ~/.claude-lab/homer/secrets/gbrain.token)
printf '%s' "$TOKEN" | shasum -a 256

# А вот так считать НЕЛЬЗЯ (включает \n)
shasum -a 256 ~/.claude-lab/homer/secrets/gbrain.token   # WRONG hash (includes \n)
```

**Фикс:**

```bash
# На VPS, в psql:
psql -d gbrain -c "SELECT agent, substring(token_sha256, 1, 16) FROM agent_tokens;"

# Сверь с тем, что считает printf '%s' "$TOKEN" | shasum -a 256
# Если расходятся – пересохрани токен без newline:
printf '%s' "<your-bearer-token>" > ~/.claude-lab/homer/secrets/gbrain.token
```

## 2. MCP-сервер не подключается

**Симптом:** Claude Code не показывает `mcp__gbrain-*` tools в списке, либо при вызове отвечает «tool not found».

**Причины:**

- Кривой синтаксис `.mcp.json` (запятая, скобка)
- Файл лежит не в той директории
- Claude Code не перезапущен после правки

**Проверка:**

```bash
# Синтаксис
jq . ~/.claude-lab/homer/.claude/.mcp.json
# Если jq падает – правь JSON

# Путь
ls -la ~/.claude-lab/homer/.claude/.mcp.json
# Должен быть .mcp.json (с точкой) внутри .claude

# Перезапуск
# Полностью выйди из Claude Code (Ctrl+D), запусти заново
```

**Дополнительно:** Claude Code чувствителен к рабочей директории. Запускай `claude` именно из `~/.claude-lab/<agent>/.claude/`, иначе он ищет `.mcp.json` в текущей папке и не находит.

## 3. recall пусто, хотя только что записал

**Симптом:** `mcp__gbrain-memory__create_decision_note` отработал, `mcp__gbrain-recall__recall` возвращает 0 matches.

**Причины:**

- Эмбеддинг ещё не построен (FastEmbed работает асинхронно)
- Query слишком далёк по смыслу от записанного текста
- Запись прошла в один scope, recall ищет в другом

**Проверка:**

```bash
# На VPS – посмотри, что реально записалось
ls /opt/gbrain/vault/30-decisions/text/ | tail -5

# В psql – проверь embedding
psql -d gbrain -c "SELECT id, agent, has_embedding FROM memory_notes ORDER BY created_at DESC LIMIT 5;"
```

**Фикс:**

- Подожди 30-60 секунд после записи – embedding догонит
- Попробуй query ближе к тексту: если в decision слово «Next.js», ищи `recall(query="Next.js")`
- Если `has_embedding=false` через минуту – проверь логи `journalctl -u gbrain-recall-mcp`

## 4. swarm.notify retries бесконечно

**Симптом:** notify возвращает `status: queued`, но через минуту delivery всё ещё `queued`, не `delivered`/`acked`.

**Причина:** worker-процесс на сервере не успевает обрабатывать очередь или упал.

**Проверка на VPS:**

```bash
systemctl status gbrain-swarm-worker
journalctl -u gbrain-swarm-worker -n 50

# Очередь
psql -d gbrain -c "SELECT status, count(*) FROM swarm_outbox GROUP BY status;"
```

**Фикс:**

```bash
# Перезапуск worker
systemctl restart gbrain-swarm-worker

# Если worker не возвращается – посмотри, что в /etc/gbrain/secrets.env
# и есть ли DATABASE_URL
```

В Path A установке worker нужен только для webhook-доставки. Если webhooks не используешь, можешь его отключить – тогда notify будет лежать в outbox, но агенты увидят через `list_my_pending` (pull-модель), а не через push.

## 5. HMAC mismatch

**Симптом:** в gbrain логах `HMAC validation failed`, агент не может писать.

**Причина:** некоторые scopes требуют HMAC-подписи дополнительно к Bearer. Секрет HMAC должен быть синхронизирован между агентом и сервером.

**Проверка:**

```bash
# На сервере
grep HMAC_SECRET /etc/gbrain/secrets.env

# На клиенте (в скриптах, которые шлют HMAC)
grep HMAC ~/.claude-lab/homer/secrets/
```

**Фикс:** в Path A установке Claude Code автоматически синхронизирует. Если рассинхронизировалось – перевыпусти секрет на сервере и положи новое значение в `~/.claude-lab/<agent>/secrets/hmac.key`.

Для большинства повседневных операций (recall, notify, memory) HMAC не нужен – хватает Bearer. HMAC включают для security-sensitive endpoints (например, удаление, перевыпуск токенов).

## 6. Telegram bot не отвечает

**Симптом:** написал боту в ЛС – тишина. Или: написал в группу с упоминанием – тишина.

**Причины:**

- Privacy mode выключен у группового бота
- Токен не загружен в плагин/мост
- Telegram-мост не запущен (если используешь `dashi-plugin-claude-code` – он отдельный процесс)

**Проверка:**

```bash
# Сам токен валиден?
TOKEN=$(cat ~/.claude-lab/marketer/secrets/telegram-bot-token)
curl -s "https://api.telegram.org/bot${TOKEN}/getMe" | jq .
# Должен ответить {"ok": true, "result": {...}}
```

**Privacy:** `@BotFather` → `/mybots` → твой бот → `Bot Settings` → `Group Privacy` → **Enable** (включить). Без этого бот реагирует на каждое сообщение в группе, что плохо.

**Мост:** на Telegram-мосту не останавливаемся в этом уроке. После урока подключи `dashi-plugin-claude-code` – это отдельный туториал в его репо.

## 7. Identity mismatch (два агента видят один инбокс)

**Симптом:** Edith и Marketer в `list_my_pending` видят одни и те же сообщения, хотя имена разные.

**Причина:** в `.mcp.json` у обоих стоит один и тот же Bearer-токен. Token → identity на сервере по таблице `agent_tokens`.

**Проверка:**

```bash
# Сравни токены
sha256sum ~/.claude-lab/edith/secrets/gbrain.token
sha256sum ~/.claude-lab/marketer/secrets/gbrain.token
# Должны быть разные

# Сравни в .mcp.json
grep Bearer ~/.claude-lab/edith/.claude/.mcp.json
grep Bearer ~/.claude-lab/marketer/.claude/.mcp.json
# Должны быть разные значения
```

**Фикс:** если случайно скопировал один токен в оба – перегенерируй на VPS, разнеси правильные значения по нужным workspace, обнови `.mcp.json`.

## 8. Claude Code не видит .mcp.json

**Симптом:** запускаешь Claude Code, в списке tools нет `mcp__gbrain-*`. При попытке вызвать – «tool not found».

**Причины:**

- Working directory не та (Claude Code запущен не из `<workspace>/.claude/`)
- Опечатка в имени файла (`mcp.json` вместо `.mcp.json`)
- Конфликт с глобальным `~/.mcp.json` (если ты экспериментировал)

**Проверка:**

```bash
# Где ты?
pwd

# Файл точно там?
ls -la $(pwd)/.mcp.json

# Глобальный mcp.json есть?
ls -la ~/.mcp.json ~/.claude/mcp.json 2>&1
```

**Фикс:** всегда запускай Claude Code командой:

```bash
cd ~/.claude-lab/<agent>/.claude
claude
```

Не из `~`, не из `~/projects`, не из любого другого места. Только из самой `.claude` папки нужного агента.

## Если ничего не помогает

1. Проверь, что VPS жив: `ssh root@your-server && uptime`
2. Проверь четыре сервиса: `systemctl status 'gbrain-*'`
3. Проверь Caddy: `systemctl status caddy` + `journalctl -u caddy -n 30`
4. Проверь Postgres: `systemctl status postgresql` + `psql -d gbrain -c "SELECT count(*) FROM memory_notes;"`
5. Открой issue в `qwwiwi/agentos-skills` с конкретными логами – уроки обновляются по реальным сбоям

## Готово

Если дочитал и почти всё работает – ты прошёл урок. Возвращайся к [README урока](./README.md), читай секцию «Что делать после урока» – там идеи для следующего шага.
