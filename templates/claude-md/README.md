# CLAUDE.md templates

Три готовых шаблона `CLAUDE.md` для запуска агентов на Claude Code. Каждый шаблон – рабочий identity-файл, который Claude Code подтягивает в контекст при старте сессии. Заменить плейсхолдеры – и агент готов к работе.

## Что есть

- **coder.md** – кодер / архитектор / ревьюер. Пишет код, проектирует системы, охраняет инфраструктуру, делает code review с двойной моделью.
- **marketer.md** – маркетолог / контент-креатор. Ведёт каналы (Telegram, Instagram, YouTube, TikTok, X, email), занимается лидогенерацией, считает unit-экономику.
- **inbox-monitoring.md** – Second Brain (Karpathy-style) и мониторинг группового чата. Автоматически сохраняет всё входящее в raw/, компилирует в wiki/, отвечает в чатах участников.

## Placeholders

Все шаблоны используют `{{двойные фигурные скобки}}` как точки замены. Прежде чем заливать шаблон в workspace агента, замени их на свои значения.

| Placeholder | Что значит | Пример значения |
|---|---|---|
| `{{agent_name}}` | ID агента, kebab-case, латиница | `homer`, `nova`, `atlas` |
| `{{Имя агента}}` | Display-имя для приветствий и SOUL-блока | «Хомер», «Нова», «Атлас» |
| `{{обращение к владельцу}}` | Как агент обращается к владельцу | «принц», «босс», «вождь», «капитан» |
| `{{владельца}}` | Родительный падеж от роли владельца | «принца», «босса», «вождя» |
| `{{владелец}}` | Именительный падеж владельца | «принц», «босс», «вождь» |
| `{{Владелец}}` | То же с заглавной буквы (начало предложения) | «Принц», «Босс», «Вождь» |
| `{{my_bot}}` | Username Telegram-бота этого агента | `myagent_bot`, `coderbot_team`, `marketingaibot` |
| `{{my_name}}` | Имя агента в swarm / tasks (обычно совпадает с `{{agent_name}}`) | `homer`, `edith`, `marketer` |
| `{{tov_source_path}}` | Путь к источнику Tone-of-Voice | `gbrain-memory:tov/main`, `core/TONE_OF_VOICE.md` |
| `{{coder}}` | ID агента-кодера в команде | `homer`, `dev-agent` |
| `{{secondbrain}}` | ID агента-Second Brain | `librarian`, `archive` |
| `{{crm}}` | ID агента-CRM | `atlas`, `sales-agent` |
| `{{coordinator}}` | ID координатора команды | `lily`, `chief` |
| `{{gbrain_host}}` | Хост твоего self-hosted gbrain | `mcp.example.com`, `gbrain.mycompany.io` |
| `{{my_namespace}}` | Namespace в long-term store (для monitoring) | `second-brain-main`, `inbox-prod` |
| `{{store_root}}` | Корневой путь Second Brain хранилища | `~/second-brain`, `/data/wiki` |
| `{{long-term store}}` | Описание системы хранения (произвольный текст) | «локальная wiki + векторный индекс», «Obsidian-vault на NAS» |
| `{{целевой группы}}` | Какой групповой чат мониторим | «чата участников курса», «общего рабочего чата» |
| `{{support-skill}}` | Имя скилла, который отвечает на вопросы участников | `course-support`, `faq-bot` |
| `{{Shared memory / runtime store}}` | Описание твоего runtime-хранилища (тоже произвольный текст) | «gbrain shared layer», «Postgres + pgvector» |
| `{{языке владельца}}` | Язык commit-сообщений | «русском», «английском» |
| `{{product}}` | Название твоего продукта (если упоминается) | «MyCourse», «AgentLab» |

Не все плейсхолдеры встречаются во всех шаблонах – заменяй то, что реально есть в выбранном файле.

## gbrain integration

Все три шаблона рассчитаны на работу с self-hosted **gbrain** – это связка из 4 MCP-серверов поверх Postgres + pgvector. Если у тебя ещё нет своего gbrain, посмотри отдельный гайд по разворачиванию (в `docs/` корневого репо).

Четыре сервера в `.mcp.json`:

- **`gbrain-memory`** – структурированные ноты (decisions, runbooks, error-patterns, external).
- **`gbrain-recall`** – семантический поиск по shared brain команды.
- **`gbrain-swarm`** – inbox для inter-agent коммуникации (notify / ack / broadcast).
- **`gbrain-tasks`** – задачи, статусы, agent heartbeat.

### Как шаблоны используют gbrain

**Recall перед действием.** Coder делает `gbrain-recall.recall(query)` перед написанием нетривиального кода – вдруг в команде уже был похожий патч или error-pattern. Marketer вызывает recall перед написанием поста – вдруг тема уже была, формат выгорел или есть свежие данные от CRM-агента. Monitoring сначала ищет в gbrain, и только потом – в локальной wiki.

**Dual-write для решений.** Когда агент сохраняет важное решение в `core/warm/decisions.md`, он параллельно вызывает `gbrain-memory.create_decision_note(...)`. Локальный markdown – canonical копия, gbrain – shared слой для всей команды. Идемпотентность по sha256 – повторный вызов с тем же body не создаёт дубликата.

**Swarm для inter-agent.** Один агент зовёт другого ТОЛЬКО через `gbrain-swarm.notify(to_agent="X", payload={...})`. Никогда не через чужие Telegram-боты, никогда не напрямую в Bot API. Это даёт outbox state machine с retry, audit log и единую точку контроля.

**Tasks для координации.** Boot sequence в начале каждой сессии:

```
gbrain-swarm.list_my_pending(agent="{{my_name}}")
gbrain-tasks.task_list(assignee="{{my_name}}", status="new")
gbrain-tasks.task_list(assignee="{{my_name}}", status="progress")
gbrain-tasks.agent_heartbeat(agent="{{my_name}}", status="online")
```

Без `agent_heartbeat` координатор будет считать агента offline. Heartbeat нужно повторять при долгих сессиях (раз в 10–15 минут).

**Dual-report.** Когда агент выполнил задачу от другого агента (пришла через swarm), он не только отвечает владельцу в Telegram, но и пишет короткое резюме координатору через `gbrain-swarm.notify(to_agent="{{coordinator}}", payload=...)` – иначе координатор не видит работы и теряет контекст для follow-up.

## Как использовать

1. **Склонируй репо** (или вытащи нужный шаблон через `curl`):

   ```bash
   git clone https://github.com/qwwiwi/agentos-skills.git
   cd agentos-skills/templates/claude-md/
   ```

2. **Скопируй шаблон в workspace будущего агента**:

   ```bash
   mkdir -p ~/.claude-lab/my-coder/.claude
   cp coder.md ~/.claude-lab/my-coder/.claude/CLAUDE.md
   ```

3. **Замени плейсхолдеры**. Открой файл, пройдись по `{{...}}` (поиском через `grep -o '{{[^}]*}}' CLAUDE.md | sort -u`), подставь свои значения.

4. **Перезапусти Claude Code** в этом workspace:

   ```bash
   cd ~/.claude-lab/my-coder
   claude
   ```

   Claude Code подтянет `CLAUDE.md` и будет вести себя по правилам шаблона.

После этого минимально нужно создать `core/USER.md`, `core/rules.md` и `.mcp.json` с подключением к твоему gbrain. Шаблоны под эти файлы можно добавлять отдельно – посмотри в `docs/` или сделай свои.
