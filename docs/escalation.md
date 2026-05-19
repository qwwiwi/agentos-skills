# Escalation — что делать когда что-то сломалось

> Когда агент не отвечает, gbrain отдаёт 401, или Telegram гейтвей замолчал —
> найди симптом в таблице ниже, открой соответствующий раздел.

## Quick triage

| Симптом | Скорее всего сломано | Раздел |
|---------|----------------------|--------|
| Telegram-бот не отвечает на сообщения | Gateway | [#gateway](#gateway) |
| `recall` возвращает 401 / 403 | gbrain auth (token mismatch) | [#gbrain-auth](#gbrain-auth) |
| `recall` возвращает 500 / timeout | gbrain server | [#gbrain-server](#gbrain-server) |
| Агент видит сообщения но не отвечает | Claude Code session | [#agent](#agent) |
| `swarm.notify` уходит, но receiver не видит | Outbox / worker | [#swarm-worker](#swarm-worker) |
| Embedding запросы зависают | Ingest worker | [#ingest-worker](#ingest-worker) |

---

## <a name='gateway'></a>Gateway не отвечает

Стек:

- `claude-tg-gateway.service` (systemd unit)
- Listens on `127.0.0.1:9090` (webhook target)
- Reverse proxy (Caddy / nginx) терминирует TLS перед ним

Шаги диагностики (в порядке):

```bash
# 1. Жив ли сам gateway?
sudo systemctl status claude-tg-gateway

# 2. Слушает ли порт?
sudo ss -tlnp | grep 9090

# 3. Telegram действительно посылает webhook?
sudo journalctl -u claude-tg-gateway -n 100 --no-pager

# 4. Если webhook регистрация сбилась — переустановить
curl -s "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"
# url должен указывать на свой домен / IP

# 5. Reverse proxy жив?
sudo systemctl status caddy   # или nginx
sudo journalctl -u caddy -n 50
```

Возможные причины + фикс:

- **Gateway не запустился** — `journalctl` покажет ошибку. Часто: неверный path в EnvironmentFile, отсутствующий config.json.
- **Cert expired** — `curl -v https://yourbot.example.com/webhook` → ошибка сертификата. Caddy auto-renew должен работать; если нет — `systemctl restart caddy`.
- **Webhook URL устарел** (например, IP сменился) — переустановить:
  ```bash
  curl -s -X POST "https://api.telegram.org/bot<TOKEN>/setWebhook" \
    -d "url=https://yourbot.example.com/webhook"
  ```
- **Process zombie** — `systemctl restart claude-tg-gateway`. Если перезапуски не помогают — оператор должен решить (gateway трогать без OK — RED zone).

Откуда взять стек:

- `jarvis-telegram-gateway` — до 2026-06-15. Repo: https://github.com/qwwiwi/jarvis-telegram-gateway
- `dashi-plugin-claude-code` — после 2026-06-15 cutover. Repo: https://github.com/qwwiwi/dashi-plugin-claude-code

---

## <a name='gbrain-auth'></a>gbrain auth (401/403)

Шаги:

```bash
# 1. Какой токен использует агент?
grep -E '"Authorization"' ~/.claude-lab/<agent>/.claude/.mcp.json

# 2. Этот sha256 в таблице agent_tokens?
sudo -u gbrain psql gbrain -c \
  "SELECT agent, can_write_scopes FROM agent_tokens \
   WHERE token_sha256 = encode(digest('<BEARER>', 'sha256'), 'hex');"

# 3. Скоупы соответствуют операции?
# Если агент пытается create_decision_note но не имеет '30-decisions'
# в can_write_scopes — 403
```

Фикс:

- **Token mismatch** — заново сгенерировать токен через скрипт из `public-gbrain-agentos`, обновить `agent_tokens.token_sha256`, обновить `.mcp.json` агента.
- **Scope mismatch** — добавить нужный scope:
  ```sql
  UPDATE agent_tokens
  SET can_write_scopes = array_append(can_write_scopes, '<scope>')
  WHERE agent = '<agent>';
  ```
- **Trailing newline в token** — при хешировании bearer не должно быть `\n`. Если sha256 файла с токеном не совпадает с тем что в БД — скорее всего `cat file` добавил перевод строки. Используй `printf '%s' "$TOKEN"` или `tr -d '\n'`.

---

## <a name='gbrain-server'></a>gbrain server (5xx / timeout)

3 MCP сервиса + 2 worker'а — все systemd units. Проверка:

```bash
sudo systemctl status gbrain-swarm gbrain-memory gbrain-recall \
                      gbrain-ingest-worker gbrain-swarm-worker
```

Если что-то крашится:

```bash
# Логи
sudo journalctl -u gbrain-memory -n 200 --no-pager

# Postgres жив?
sudo systemctl status postgresql

# pgvector установлен?
sudo -u gbrain psql gbrain -c \
  "SELECT extname FROM pg_extension WHERE extname = 'vector';"

# embedding worker не блокирует очередь?
sudo -u gbrain psql gbrain -c \
  "SELECT COUNT(*) FROM embedding_jobs WHERE status = 'pending';"
```

Restart procedure:

```bash
# Restart всех 3 MCP + 2 worker за один шаг
sudo systemctl restart 'gbrain-*'

# Через 5 секунд — smoke
curl -H "Authorization: Bearer <YOUR_TOKEN>" \
  http://127.0.0.1:8768/recall/mcp \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Если Postgres не стартует — проверь disk space (`df -h`). Самая частая причина 500 на gbrain — закончилось место под WAL.

---

## <a name='agent'></a>Агент не отвечает

Симптомы: gateway получает webhook, журнал показывает spawn, но Claude Code не выдаёт ответа.

Причины:

1. **Anthropic API key expired** или **rate limit** — проверь:
   ```bash
   curl -H "x-api-key: <YOUR_KEY>" https://api.anthropic.com/v1/models
   ```
   401 — ключ невалиден. 429 — упёрся в лимит, ждать ресета.
2. **Claude Code зациклился в MCP** — журнал `journalctl -u claude-tg-gateway` покажет «MCP server unresponsive» или timeout. Поправить: restart MCP сервера + перезапустить агента (`POST /restart-agent/<agent>` в gateway).
3. **Workspace битый** (missing `CLAUDE.md`, missing `.mcp.json`) — Claude Code упадёт сразу при старте. Проверка:
   ```bash
   claude --workspace ~/.claude-lab/<agent>/.claude/ --help
   ```
4. **`hot/handoff.md` разросся** — если файл > 50KB, контекст агента перегружен. Прогнать ротацию:
   ```bash
   ~/.claude-lab/<agent>/.claude/scripts/rotate-handoff.sh
   ```

---

## <a name='swarm-worker'></a>swarm-worker

Что: drain'ит `delivery_outbox`, POST'ит inter-agent message в gateway получателя.

Если получатель не видит сообщения:

```bash
# 1. Outbox пустой?
sudo -u gbrain psql gbrain -c \
  "SELECT id, from_agent, to_agent, status, attempts \
   FROM delivery_outbox WHERE status != 'acked' \
   ORDER BY id DESC LIMIT 20;"

# 2. Worker жив?
sudo systemctl status gbrain-swarm-worker

# 3. Логи (последние 10 попыток)
sudo journalctl -u gbrain-swarm-worker -n 50

# 4. Получатель достижим?
curl -X POST http://127.0.0.1:9090/webhook -d '{"test":"ping"}'
```

Если `attempts` растёт, а `status = pending` — worker пытается, но получатель не отвечает. Проверь gateway получателя по разделу [#gateway](#gateway).

---

## <a name='ingest-worker'></a>ingest-worker (embeddings)

Что: тянет jobs из `embedding_jobs`, embeds через multilingual-e5-large (FastEmbed, CPU), пишет в `chunks`.

Если recall не находит свежие документы:

```bash
sudo -u gbrain psql gbrain -c \
  "SELECT status, COUNT(*) FROM embedding_jobs GROUP BY status;"
# Should be mostly 'done', few 'pending', zero 'failed'

# Если pending растёт — worker не успевает
sudo systemctl status gbrain-ingest-worker
sudo journalctl -u gbrain-ingest-worker -n 100
```

Часто причина — FastEmbed модель не загружена (не хватило RAM при первом запуске). Restart + проверить `journalctl` на «model loaded».

Если `failed` ≥ 1 — взять конкретный job и посмотреть ошибку:

```bash
sudo -u gbrain psql gbrain -c \
  "SELECT id, error FROM embedding_jobs WHERE status = 'failed' LIMIT 5;"
```

---

## Эскалация оператору

Если ни один из шагов не помог за 15 минут — отправь в Telegram оператору сообщение по шаблону:

```
Симптом: <что не работает>
Что пробовал: <шаги из этого документа>
Что нашёл в логах: <вырезка>
Команда диагностики (для воспроизведения): <команды>
```

Координатор-агент (Homer) часто может помочь, но финальное решение всегда за оператором.

---

## Полезные ссылки на соседние репо

- `qwwiwi/public-gbrain-agentos` — исходный код gbrain MCP + workers
- `qwwiwi/jarvis-telegram-gateway` / `qwwiwi/dashi-plugin-claude-code` — gateway / channel plugin
- `qwwiwi/public-architecture-claude-code` — workspace generator + hooks (handoff, ротация)
