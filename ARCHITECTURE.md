# Архитектура репо `agentos-skills`

Одностраничный обзор того, как этот репо устроен и как он связан со своими соседями – четырьмя другими публичными репозиториями.

## Дерево репозитория

```
agentos-skills/
├── README.md
├── ARCHITECTURE.md          (этот файл)
├── LICENSE
│
├── senior-brainstorm/                 (existing skill)
├── telegram-bot-builder/              (existing skill)
├── agentos-skills-navigator/          (новый skill – навигация по репо)
│
├── templates/
│   ├── README.md
│   ├── 3-claude-md-templates-v2.html
│   └── claude-md/
│       ├── README.md
│       ├── coder.md
│       ├── marketer.md
│       └── inbox-monitoring.md
│
├── lessons/
│   ├── README.md
│   └── lesson-3-agents-with-gbrain/
│       ├── README.md
│       ├── 01-overview.md
│       ├── 02-prerequisites.md
│       ├── 03-create-coder.md
│       ├── 04-create-marketer.md
│       ├── 05-create-sales.md
│       ├── 06-setup-gbrain.md
│       ├── 07-connect-agents.md
│       ├── 08-test-coordination.md
│       └── 09-troubleshooting.md
│
├── architecture/
│   ├── README.md
│   ├── 01-claude-code-arch.md
│   ├── 02-gbrain-shared-memory.md
│   ├── 03-telegram-bridge.md
│   ├── 04-claude-md-guide.md
│   └── diagram.md
│
└── docs/
    ├── repo-registry.md
    ├── prerequisites.md
    └── faq.md
```

## Уровни абстракции

Содержимое репо делится на три уровня – от самого конкретного к самому общему:

1. **Конкретика** – `templates/`, `lessons/`. То, что копируешь и запускаешь руками. Файлы здесь работают «из коробки».
2. **Концепции** – `architecture/`, `agentos-skills-navigator/`. Как думать о системе, какой репо за что отвечает, как они соединяются.
3. **Skills** – `senior-brainstorm/`, `telegram-bot-builder/`. Готовые ассистенты для частных задач (архитектура, telegram-боты).

Учиться удобно сверху вниз: сначала прочитать архитектуру, потом пройти урок, потом разворачивать собственных агентов на templates.

## Связь с соседними публичными репо

Этот репо – одна из пяти публичных частей AgentOS. Каждая часть отвечает за свой слой.

| Репо | Для чего нужен | Когда подключить |
|---|---|---|
| [`qwwiwi/agentos-skills`](https://github.com/qwwiwi/agentos-skills) | Точка входа: skills, templates, уроки, архитектурные конспекты | Сразу – с него начинается интенсив |
| [`qwwiwi/public-architecture-claude-code`](https://github.com/qwwiwi/public-architecture-claude-code) | Workspace generator: создаёт изолированный workspace на каждого агента | На шаге создания первого агента (lesson-3, шаги 03–05) |
| [`qwwiwi/public-gbrain-agentos`](https://github.com/qwwiwi/public-gbrain-agentos) | Shared memory: Postgres + pgvector + 4 MCP-сервера (memory / recall / swarm / tasks) | На шаге setup gbrain (lesson-3, шаг 06) |
| [`qwwiwi/dashi-plugin-claude-code`](https://github.com/qwwiwi/dashi-plugin-claude-code) | Telegram bridge: соединяет Claude Code с Telegram-ботом агента | Когда хочешь общаться с агентом через Telegram, а не только через CLI |
| [`qwwiwi/edgelab-claude-md`](https://github.com/qwwiwi/edgelab-claude-md) | Расширенный гайд по CLAUDE.md (анатомия из 18 элементов) | Когда подгоняешь template под свои сценарии и хочешь понять, что и почему |

## Поток ученика

Типичный путь от нуля до работающей тройки агентов:

1. Клонирует `agentos-skills` (этот репо).
2. Читает `README.md` + `ARCHITECTURE.md` – понимает, где что лежит.
3. Открывает `lessons/lesson-3-agents-with-gbrain/` и идёт по шагам.
4. По уроку устанавливает четыре публичных репо:
   - три раза `public-architecture-claude-code` (workspace на coder, marketer, sales)
   - один раз `public-gbrain-agentos` (общая память)
   - три раза `dashi-plugin-claude-code` (по Telegram-боту на агента, опционально)
5. Берёт CLAUDE.md template из `templates/claude-md/` под роль каждого агента, заменяет placeholders.
6. Подключает skill `agentos-skills-navigator` – дальше Claude Code сам подсказывает, какой файл репо открыть.
7. Smoke-тест: три агента видят общий gbrain и обмениваются сообщениями через `swarm.notify`.

## Что НЕ входит в этот репо

Чтобы не было сюрпризов: следующего тут нет и не будет:

- Реальные токены, секреты, инстансы gbrain или БД – только примеры, placeholders, инструкции по созданию своих.
- Чужие приватные репозитории (mapping-репо, внутренние реестры и т.п.) – только публичные ссылки.
- Кастомные интеграции под конкретные продукты (готовые боты со своей бизнес-логикой) – на это есть отдельные приватные репо у каждой команды.

Этот репо – база. Дальше каждый ученик собирает свой стек: какие агенты ему нужны, какие skills подключить, какие интеграции дописать.
