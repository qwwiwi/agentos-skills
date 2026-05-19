# Layer 2: Shared memory (gbrain)

Репо: [`qwwiwi/public-gbrain-agentos`](https://github.com/qwwiwi/public-gbrain-agentos)
Лицензия: Apache 2.0

## Что это

Second Brain для команды AI-агентов. Self-hosted сервис: разворачиваете его на своём VPS, и все ваши агенты получают общую память. Один агент пишет «решили использовать Postgres вместо MongoDB» – другой через минуту видит это решение в recall.

Без gbrain у каждого агента память изолирована – он не знает, что вчера решил его «коллега» в соседнем workspace. С gbrain это перестаёт быть проблемой.

## Стек

- **Postgres + pgvector** – хранилище нот и embeddings
- **FastEmbed multilingual-e5-large** – локальные embeddings (без OpenAI)
- **4 MCP сервера** – интерфейс для агентов
- **Hermes inbox** – Telegram-канал для долгосрочных заметок
- **Vault** – markdown-структура внутри Postgres (можно экспортировать)

## 4 MCP сервера

Каждый сервер – отдельный endpoint, агент подключается через стандартный MCP-клиент Claude Code.

| Сервер | Роль | Основные tools |
|---|---|---|
| **memory** (write) | Сохранение нот | `create_decision_note`, `create_runbook_note`, `create_error_pattern_note` |
| **recall** (read) | Поиск по shared brain | `recall(query)`, `get(uri)`, `related(uri)` |
| **swarm** (inter-agent) | Сообщения между агентами | `notify(to_agent)`, `list_my_pending`, `ack` |
| **tasks** (board) | Координация задач | `task_create`, `task_list`, `task_start`, `task_review`, `task_done` |

Разделение на 4 сервера – не для красоты. Это RBAC: можно дать одному агенту полный доступ к memory, а другому – только recall. И аудит-логи разносятся аккуратно по доменам.

## Path A vs Path B

При установке вы выбираете один из двух путей.

**Path A: минимальный.**
Только сервер на VPS, без авто-генерации workspace для агентов. Подходит, если у вас уже есть свои workspace (например, через `public-architecture-claude-code`) и нужен только backend.

**Path B: с генератором workspace.**
Сервер + интерактивный генератор: пройдёте 8–12 вопросов, и установщик создаст готовые workspace для нужного числа агентов и подключит их к gbrain. Подходит для нулевой установки.

Большинство выбирают Path A, потому что workspace удобнее ставить через install.sh из Layer 1 – он более гибкий.

## HMAC vs Bearer auth

Сервер поддерживает два режима аутентификации:

- **Bearer** – простой токен в `Authorization: Bearer <token>`. Подходит для локальных агентов внутри Tailscale-сети.
- **HMAC** – подпись запросов с timestamp. Подходит для публичных endpoints (когда сервер торчит наружу).

В сетапе по умолчанию – Bearer + reverse proxy через Caddy с TLS. Этого хватает для 95% сценариев.

## Деплой

Один VPS на 4 vCPU / 8 GB RAM (Hetzner CPX42 или аналог). Время установки – 30–90 минут в зависимости от того, насколько вы знакомы со стеком.

```bash
git clone https://github.com/qwwiwi/public-gbrain-agentos.git
cd public-gbrain-agentos
# далее по AGENT.md – даёте файл свежему Claude Code,
# он проводит вас через установку
```

После установки у вас:
- 4 MCP сервера на `https://<your-domain>/{memory,recall,swarm,tasks}/mcp`
- Bearer-токены на каждого агента
- Hermes inbox-агент (опционально)

## Когда использовать

| Сценарий | Подходит? |
|---|---|
| 2 и больше агентов с общими задачами | да, основной use case |
| Нужен долгосрочный inbox через Telegram | да, Hermes для этого |
| Один локальный агент | избыточно – Layer 1 хватит |
| Просто блокнот для заметок | избыточно – обычный markdown быстрее |
| Команда из 5+ агентов на разных хостах | да, ради этого и сделано |

## Когда НЕ нужно

- Один агент. Локальной памяти Layer 1 хватает с запасом.
- Нет VPS и не планируется. Cloud-hosted альтернатив сейчас нет.
- Нужна only-read база знаний. Тогда проще obsidian vault + grep.

## Как агенты подключаются

В `.mcp.json` workspace каждого агента прописывается:

```json
{
  "mcpServers": {
    "gbrain-memory": {
      "url": "https://<your-domain>/memory/mcp",
      "headers": {"Authorization": "Bearer <agent-token>"}
    },
    "gbrain-recall": { "url": "https://<your-domain>/recall/mcp", ... },
    "gbrain-swarm":  { "url": "https://<your-domain>/swarm/mcp",  ... },
    "gbrain-tasks":  { "url": "https://<your-domain>/tasks/mcp",  ... }
  }
}
```

Дальше агент видит инструменты `mcp__gbrain-memory__create_decision_note` и работает с ними как с обычными tools.

## Совместимость

- Совместим с любым MCP-клиентом, не только Claude Code (через стандарт MCP 2025-06-18).
- Совместим с Layer 1 workspace из коробки.
- Совместим с Layer 3 Telegram bridge: Hermes inbox + plugin-channel = end-to-end inbox-чат.

## Внешняя ссылка

Документация, установщик, AGENT.md: https://github.com/qwwiwi/public-gbrain-agentos
