# 01. Обзор: зачем три агента и общий gbrain

## Зачем нужны три агента

Один Claude Code – универсальный помощник. Но у него нет специализации, у него нет роли. Каждый раз ты объясняешь контекст заново: «я сейчас пишу код», «а теперь принимаем ссылку в Second Brain», «а теперь делаем пост».

Три агента решают это разделением ответственности. По правилу «build / remember / grow» – минимальная полезная команда для одного оператора.

### Homer – build (coder / architect / coordinator)

- Пишет код, чинит баги, делает рефакторинг
- Знает твой стек (TypeScript / Python / Go – что используешь)
- Соблюдает твои правила (форматирование, тесты, code review)
- Владеет общей инфраструктурой: gateway, gbrain-конфиг, deploys
- Координирует двух других агентов (роль tech lead)
- Workspace: `~/.claude-lab/homer/.claude/`

### Edith – remember (Second Brain / inbox / ingest)

- Ловит каждую ссылку / voice / файл / сниппет, который кидаешь команде
- Pipeline: **raw → wiki → output** (метод Karpathy Second Brain)
- Имеет собственное локальное дерево знаний помимо gbrain:
  - `raw/` – immutable inbox (YAML-frontmatter markdown)
  - `wiki/` – LLM-compiled страницы (sources, entities, concepts, synthesis)
  - `output/` – генерируемые артефакты по запросу
- Параллельно дублирует summary всех ingest'ов в gbrain, чтобы остальные агенты могли recall'ить
- Workspace: `~/.claude-lab/edith/.claude/`

### Marketer – grow (контент / lead-gen / public presence)

- Владеет публичным Instagram-присутствием: пишет reels, hooks, captions, scripts
- Скрапит референсные аккаунты (например healthy-breakfast niche 100k+) через Apify / HikerAPI / Cobalt / Whisper
- Держит TOV (tone of voice) canonical snapshot и отказывается публиковать без него
- Готовит сценарии для видео, презентации, лендинги
- Не пишет production-код – не его роль
- Workspace: `~/.claude-lab/marketer/.claude/`

Каждый агент в своём workspace, со своим CLAUDE.md, своим набором скиллов, своим Telegram-ботом. Не путаются контексты.

> Добавить четвёртого агента (например `sales`) – ~5 минут: создать workspace из шаблона, минтнуть Telegram-бот, выпустить gbrain-токен, добавить запись в gateway-конфиг, перезапустить gateway. Система рассчитана на расширение через добавление, а не через переписывание.

## Почему shared gbrain

Без общей памяти агенты – острова. Homer написал решение по архитектуре, но Marketer его не видит. Marketer запустил рекламу, а Edith не знает про новый поток ссылок-источников. Каждый раз ты сам становишься «шиной данных» – пересказываешь одному что сделал другой.

gbrain (Second Brain команды) решает это:

- **memory** – хранит decisions, runbooks, error-patterns, external notes в markdown vault
- **recall** – семантический поиск по vault (FastEmbed multilingual-e5-large) + полнотекст
- **swarm** – инбокс между агентами (`notify`, `list_my_pending`, `ack`)
- **tasks** – kanban-доска задач (`task_create`, `task_list`, `task_start`)

Каждый агент пишет важное в gbrain. Каждый агент читает gbrain перед началом задачи. Контекст переживает рестарты и compact'ы.

## Как агенты взаимодействуют

Три способа коммуникации:

### 1. swarm.notify (inter-agent сообщения)

Homer заканчивает фичу и сообщает Marketer:

```text
swarm.notify(
  to_agent="marketer",
  payload={
    "title": "Лендинг готов к публикации",
    "body": "URL: example.com/feature-x. Нужен пост в Telegram"
  }
)
```

Marketer при следующем запуске видит:

```text
list_my_pending(agent="marketer")
→ [{ from: "homer", title: "Лендинг готов...", ... }]
```

Подтверждает получение: `ack(delivery_id)`.

### 2. recall (общая база знаний)

Marketer хочет понять, что Edith сохранила про конкурентов:

```text
recall(query="конкуренты healthy breakfast", limit=5)
→ возвращает 5 заметок из shared vault с релевантными ссылками
```

Не нужно помнить, кто и когда это записал – recall находит.

### 3. tasks (kanban)

Homer создаёт задачу для Marketer:

```text
task_create(
  title="Подготовь captions для нового видео",
  assignee="marketer",
  body="..."
)
```

Marketer видит свои задачи: `task_list(assignee="marketer", status="new")`.

## Архитектура на одном слайде

```
                  ┌──────────────────────────────────┐
                  │   gbrain VPS                     │
                  │   your-gbrain.example.com        │
                  │                                  │
                  │  ┌────────────┐  ┌─────────────┐ │
                  │  │ Postgres + │  │ Markdown    │ │
                  │  │ pgvector   │  │ vault       │ │
                  │  └────────────┘  └─────────────┘ │
                  │         ▲              ▲         │
                  │  ┌──────┴──────────────┴──────┐  │
                  │  │ 4 MCP servers (HTTP+TLS)   │  │
                  │  │ memory / recall / swarm /  │  │
                  │  │ tasks                      │  │
                  │  └─────────┬──────────────────┘  │
                  └────────────┼─────────────────────┘
                               │ Bearer токены
                ┌──────────────┼──────────────┐
                │              │              │
         ┌──────▼─────┐ ┌──────▼─────┐ ┌──────▼─────┐
         │   Homer    │ │   Edith    │ │  Marketer  │
         │  (build)   │ │ (remember) │ │   (grow)   │
         │            │ │            │ │            │
         │  Claude    │ │  Claude    │ │  Claude    │
         │  Code      │ │  Code      │ │  Code      │
         │  + CLAUDE  │ │  + CLAUDE  │ │  + CLAUDE  │
         │    .md     │ │    .md     │ │    .md     │
         │  + .mcp    │ │  + .mcp    │ │  + .mcp    │
         │    .json   │ │    .json   │ │    .json   │
         └────────────┘ └────────────┘ └────────────┘
              ▲              ▲              ▲
              └──────────────┼──────────────┘
                             │
                       ┌─────┴──────┐
                       │  Ты – через│
                       │  CLI или   │
                       │  Telegram  │
                       └────────────┘
```

Ключевое:

- Один gbrain на всех (centralized memory)
- Каждый агент – отдельный процесс Claude Code в своём workspace
- Связь агента с gbrain – через MCP по HTTPS с Bearer-токеном
- Связь агентов между собой – через swarm.notify, которая внутри gbrain
- Ты говоришь с агентами либо в терминале, либо через Telegram-мост

## Что дальше

Перейди к [02-prerequisites.md](./02-prerequisites.md) – проверь, что у тебя есть всё нужное.
