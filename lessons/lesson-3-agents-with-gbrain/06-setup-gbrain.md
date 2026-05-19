# 06. Поднимаем gbrain на собственном VPS

Самая инфраструктурная часть урока. Нужно завести Second Brain команды – Postgres+pgvector с четырьмя MCP-серверами поверх. Хорошие новости: для этого есть готовый дистрибутив `public-gbrain-agentos`, который сам себя устанавливает через Claude Code.

Идея: ты подключаешься к свежему VPS по SSH, клонируешь репо, передаёшь Claude Code файл AGENT.md и отвечаешь на 8-12 вопросов. Через 30-90 минут gbrain доступен по HTTPS с TLS и четырьмя MCP-серверами.

## Что делаем

1. SSH на VPS, ставим Claude Code на сервере
2. Клонируем `public-gbrain-agentos`
3. Запускаем установочный диалог через `AGENT.md`
4. Выбираем Path A (минимальный) – без Telegram-инбокса, только MCP
5. Привязываем домен, ждём TLS
6. Создаём три персональных Bearer-токена (Homer / Edith / Marketer)
7. Smoke: проверяем, что recall отвечает 200 OK

Времени: 30–90 минут (зависит от скорости интернета, опыта работы с VPS, провижининга у провайдера). Базовая установка – 30 минут, +15-60 минут на полную настройку (4 MCP-сервера + Caddy + TLS-сертификат + FastEmbed модель ~500MB + миграции Postgres).

## Шаг 1: SSH на VPS

```bash
ssh root@your-server-ip
```

Если заходит без пароля – отлично. Если просит пароль – значит SSH-ключ не положен; вернись в панель провайдера, добавь ключ.

## Шаг 2: ставим Claude Code на сервере

```bash
# На VPS
curl -fsSL https://claude.ai/install.sh | sh
claude --version   # проверяем
claude             # авторизуемся в твоём аккаунте (откроется URL, скопируй на ноут)
```

Тебе понадобится короткое окно авторизации – URL открываешь в локальном браузере, разрешаешь, копируешь обратный токен в SSH-сессию.

## Шаг 3: клонируем gbrain

```bash
# На VPS
cd /opt
git clone https://github.com/qwwiwi/public-gbrain-agentos.git
cd public-gbrain-agentos
ls   # увидишь AGENT.md, README.md, services/, ...
```

## Шаг 4: запускаем установочный диалог

```bash
# На VPS, всё ещё в /opt/public-gbrain-agentos
claude
```

В открывшейся сессии Claude Code пишешь:

```text
Прочитай AGENT.md и проведи меня через установку.
Я выбираю Path A (минимальный, без Telegram-инбокса).
Домен: your-gbrain.example.com (замени на свой).
```

Claude Code начнёт диалог. Будет задавать вопросы – отвечаешь:

| Вопрос | Ответ |
|---|---|
| Domain | `your-gbrain.example.com` |
| Email для Let's Encrypt | твой email |
| Postgres пароль | сгенерируй надёжный (24+ символов) |
| Embedding provider | `fastembed` (локально, бесплатно) |
| Embedding model | `multilingual-e5-large` (хорошо понимает русский) |
| Agent identities | `homer,edith,marketer` (через запятую) |
| Init vault | `yes` |
| Bearer-токены | сгенерируй три (по одному на агента) |

Claude Code сам поставит зависимости, поднимет Postgres, накатит миграции, сгенерирует Caddy-конфиг, запросит TLS-сертификат, поднимет 4 systemd-сервиса.

## Шаг 5: привязываем домен

Параллельно с установкой – в DNS-панели твоего домена:

- Создай A-запись: `your-gbrain` → IP твоего VPS
- TTL: 300 (5 минут)
- Если используешь Cloudflare как DNS – **отключи проксирование** (серое облако, не оранжевое)

Почему grey cloud: MCP стримит `streamable-http` (SSE). Cloudflare proxy буферизует SSE и ломает поток. Для MCP – только DNS-only.

## Шаг 6: проверяем TLS

После того как DNS прописался и Caddy получил сертификат:

```bash
# На VPS или с ноута
curl -I https://your-gbrain.example.com/
```

Ожидаешь `HTTP/2 200` (или 404 на корне – это ок, главное TLS handshake прошёл). Если `HTTP/2 200` и в headers есть `server: Caddy` – TLS живой.

## Шаг 7: создаём три Bearer-токена

В сессии Claude Code на сервере:

```text
Создай три персональных Bearer-токена в agent_tokens. Scopes – по роли агента:
- homer (coder/coordinator): 10-events, 20-daily, 30-decisions, 70-runbooks, 80-error-patterns, 90-inbox
- edith (Second Brain – inbox-heavy, external): 10-events, 20-daily, 50-external, 90-inbox
- marketer (content): 10-events, 20-daily, 60-content, 90-inbox

Выдай мне сырые токены, я скопирую и подставлю в .mcp.json агентов локально.
```

Эквивалентный SQL (Claude Code выполнит сам после миграций):

```sql
INSERT INTO agent_tokens (agent, token_sha256, can_write_scopes) VALUES
  ('homer',    '<sha256>', ARRAY['10-events','20-daily','30-decisions','70-runbooks','80-error-patterns','90-inbox']),
  ('edith',    '<sha256>', ARRAY['10-events','20-daily','50-external','90-inbox']),
  ('marketer', '<sha256>', ARRAY['10-events','20-daily','60-content','90-inbox']);
```

(Scopes иллюстративные – Edith специализируется на external ingest, Marketer на content, Homer тянет decisions / runbooks / error-patterns как координатор. По мере роста команды можно расширять.)

Claude Code сгенерирует токены, запишет sha256 в `agent_tokens`, отдаст сырые значения один раз. Скопируй и сохрани локально на ноуте:

```bash
# На ноуте
mkdir -p ~/.claude-lab/homer/secrets
mkdir -p ~/.claude-lab/edith/secrets
mkdir -p ~/.claude-lab/marketer/secrets

# Сохраняем токены (значения от Claude Code на сервере)
echo "<homer-bearer-token>"    > ~/.claude-lab/homer/secrets/gbrain.token
echo "<edith-bearer-token>"    > ~/.claude-lab/edith/secrets/gbrain.token
echo "<marketer-bearer-token>" > ~/.claude-lab/marketer/secrets/gbrain.token

chmod 600 ~/.claude-lab/{homer,edith,marketer}/secrets/gbrain.token
```

**Важно про токены:**

- Hash в Postgres = sha256 от строки токена **без trailing newline**
- Не делай `cat file | shasum` – он считает с `\n` в конце
- Если делаешь sha256 руками – `printf '%s' "$TOKEN" | shasum -a 256`
- Иначе на 1 байт разойдётся → 401 при проверке

## Шаг 8: smoke-тест recall

С ноута, используя homer-токен:

```bash
TOKEN=$(cat ~/.claude-lab/homer/secrets/gbrain.token)

curl -s -X POST https://your-gbrain.example.com/recall/mcp \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | head -30
```

Ожидаемо: 200 OK + JSON-RPC ответ со списком tools (`recall`, `get`, `recent`, `stats`, ...).

Если `401 Unauthorized` – проверь токен (см. шаг 7 про trailing newline).

Если `503` или таймаут – проверь, что Caddy и MCP-сервисы запущены:

```bash
# На VPS
systemctl status caddy
systemctl status 'gbrain-*-mcp'
```

## Что осталось из gbrain

Path A даёт минимум – память + recall + inbox + tasks. Что НЕ настраивается на этом шаге:

- **Telegram-инбокс gbrain** (Path B): bot, который собирает форварды в vault
- **Webhook-триггеры**: чтобы swarm.notify будил агента через ngrok/Tailscale
- **Бэкапы Postgres**: cron+restic, рекомендуется настроить отдельно
- **Мониторинг**: Grafana/Prometheus, опционально

Всё это можно навинтить позже. Для урока хватит Path A.

## Troubleshooting

### TLS не получается

- Проверь A-запись: `dig +short your-gbrain.example.com` – должен вернуть твой IP
- Если стоит Cloudflare proxy – выключи
- Проверь, что 80 и 443 открыты на VPS firewall: `ufw status`
- Caddy логи: `journalctl -u caddy -n 50`

### Postgres не стартует

- Часто из-за нехватки RAM (минимум 8 GiB не зря)
- `free -h` – проверь
- Если swap отсутствует, добавь 4 GB swap

### Claude Code на сервере падает

- Часто из-за нестабильного SSH (закрылся terminal – процесс убился)
- Используй `tmux` или `screen`:
  ```bash
  tmux new -s gbrain
  claude
  # Ctrl+B D – отвязаться, не убивая
  # tmux attach -t gbrain – вернуться
  ```

### Установка зависла на «installing fastembed model»

Модель ~500 MB, грузится с HuggingFace. Если медленный интернет – подожди до 10 минут. Если фейлится – проверь `df -h` (хватит ли места) и `journalctl -u gbrain-recall-mcp`.

## Готово

gbrain живой, токены сохранены. Переходи к [07-connect-agents.md](./07-connect-agents.md) – подключим три workspace к нему.
