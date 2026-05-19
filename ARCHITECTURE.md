# Архитектура репо `agentos-skills`

Карта роя плюс карта этого репо. Рой – primary. Репо – его публичная грань.

## Что мы строим

> «A team of three Claude Code agents running on a single VPS, sharing one knowledge layer (gbrain) and one inbound channel (Telegram).»
>
> – [`AGENTIC-SWARM.md`](./AGENTIC-SWARM.md), секция 1

Эти ресурсы (этот репо + четыре соседних публичных) – публичные части системы. Приватные части (реальные токены, реальные deploy-скрипты, реальный мониторинг, конкретная бизнес-логика операторов) живут в private workspace-ах операторов и не публикуются.

## Карта системы

```
        Telegram (один бот на агента)
                  │ webhook
                  ▼
        Telegram Gateway (systemd, :9090)
                  │ spawns / streams
       ┌──────────┼──────────┐
       ▼          ▼          ▼
    Homer      Edith     Marketer
   (coder)   (brain)    (growth)
       │ MCP      │ MCP      │ MCP
       └──────────┼──────────┘
                  ▼
         gbrain (shared brain)
       Postgres + pgvector + workers
       :8766 swarm  :8767 memory  :8768 recall
```

Подробный разбор каждого блока – [`AGENTIC-SWARM.md`](./AGENTIC-SWARM.md), секции 4–11.

## Что в этом репо vs что в соседних

| Компонент роя | Где описан | Где исходный код |
|---------------|------------|------------------|
| Архитектура (общий обзор) | этот репо: `AGENTIC-SWARM.md` | – |
| Workspace generator + layered memory | этот репо: `templates/claude-md/` + урок 03–05 | [`qwwiwi/public-architecture-claude-code`](https://github.com/qwwiwi/public-architecture-claude-code) |
| gbrain (Postgres + 3 MCP + workers) | этот репо: `architecture/02-gbrain-shared-memory.md` + урок 06 | [`qwwiwi/public-gbrain-agentos`](https://github.com/qwwiwi/public-gbrain-agentos) |
| Telegram gateway | этот репо: `architecture/03-telegram-bridge.md` + урок 07 | [`qwwiwi/jarvis-telegram-gateway`](https://github.com/qwwiwi/jarvis-telegram-gateway) (до 2026-06-15) / [`qwwiwi/dashi-plugin-claude-code`](https://github.com/qwwiwi/dashi-plugin-claude-code) (после) |
| CLAUDE.md гайд | этот репо: `templates/claude-md/` + `architecture/04-claude-md-guide.md` | [`qwwiwi/edgelab-claude-md`](https://github.com/qwwiwi/edgelab-claude-md) |
| Skills (engineering discipline) | этот репо + Superpowers | [`obra/superpowers`](https://github.com/obra/superpowers) |

Этот репо – единственная точка, где все пять компонентов сходятся в одну картину. Соседние репо – source of truth для своего слоя.

## Дерево репозитория

```
agentos-skills/
├── AGENTIC-SWARM.md           (главный документ – читать первым)
├── README.md
├── ARCHITECTURE.md            (этот файл)
├── LICENSE
│
├── templates/
│   ├── README.md
│   ├── 3-claude-md-templates-v2.html
│   └── claude-md/
│       ├── README.md
│       ├── coder.md                  (Homer)
│       ├── inbox-monitoring.md       (Edith – Second Brain)
│       ├── marketer.md               (Marketer)
│       └── sales.md                  (опциональный 4-й агент)
│
├── lessons/
│   ├── README.md
│   └── lesson-3-agents-with-gbrain/  (9 шагов, Homer + Edith + Marketer)
│       ├── README.md
│       ├── 01-overview.md
│       ├── 02-prerequisites.md
│       ├── 03-create-coder.md        (Homer)
│       ├── 04-create-edith.md        (Edith – Second Brain)
│       ├── 05-create-marketer.md     (Marketer)
│       ├── 06-setup-gbrain.md
│       ├── 07-connect-agents.md
│       ├── 08-test-coordination.md
│       └── 09-troubleshooting.md
│
├── architecture/              (конспекты соседних публичных репо)
│   ├── README.md
│   ├── 01-claude-code-arch.md
│   ├── 02-gbrain-shared-memory.md
│   ├── 03-telegram-bridge.md
│   ├── 04-claude-md-guide.md
│   └── diagram.md
│
├── agentos-skills-navigator/  (skill для навигации по этому репо)
│
├── docs/                      (registry, prerequisites, FAQ, escalation)
│   ├── repo-registry.md
│   ├── prerequisites.md
│   ├── faq.md
│   └── escalation.md
│
├── senior-brainstorm/         (skill для Homer – архитектурные решения)
├── senior-brainstorm.skill
├── telegram-bot-builder/      (skill для Homer/Marketer – сборка ботов)
└── telegram-bot-builder.skill
```

## Уровни абстракции

Содержимое репо делится на три уровня – от самого конкретного к самому общему:

1. **Конкретика** – `templates/`, `lessons/`. То, что копируешь и запускаешь руками. Файлы здесь работают «из коробки».
2. **Концепции** – `AGENTIC-SWARM.md`, `architecture/`, `agentos-skills-navigator/`. Как думать о системе, какой репо за что отвечает, как они соединяются.
3. **Skills** – `senior-brainstorm/`, `telegram-bot-builder/`. Готовые ассистенты для частных задач (архитектура, telegram-боты).

Учиться удобно сверху вниз: сначала `AGENTIC-SWARM.md` (что мы строим), потом `architecture/` (как соседние репо собираются вместе), потом боевой урок (Homer/Edith/Marketer руками), потом templates как референс для своих агентов.

## Поток ученика

Типичный путь от нуля до работающей тройки агентов:

1. Клонирует `agentos-skills` (этот репо).
2. Читает `AGENTIC-SWARM.md` + `README.md` + `ARCHITECTURE.md` – понимает архитектуру роя и навигацию по репо.
3. Открывает `lessons/lesson-3-agents-with-gbrain/` и идёт по шагам.
4. По уроку подключает четыре соседних публичных репо:
   - три раза `public-architecture-claude-code` (workspace на Homer, Edith, Marketer);
   - один раз `public-gbrain-agentos` (общая память + 3 MCP);
   - один раз `jarvis-telegram-gateway` (или `dashi-plugin-claude-code` после 2026-06-15) – Telegram-bridge для всех агентов;
   - `edgelab-claude-md` – референс при заполнении CLAUDE.md под свои сценарии.
5. Берёт CLAUDE.md template из `templates/claude-md/` под роль каждого агента, заменяет placeholders.
6. Ставит обязательный skill **Superpowers** (от `obra` или из Anthropic marketplace) – без него агент не следует инженерной дисциплине.
7. Подключает `agentos-skills-navigator` – дальше Claude Code сам подсказывает, какой файл репо открыть под вопрос ученика.
8. Smoke-тест: Homer, Edith и Marketer видят общий gbrain, обмениваются сообщениями через `swarm.notify`, отвечают в Telegram через свои боты.

## Что НЕ входит в этот репо

Чтобы не было сюрпризов – следующего тут нет и не будет:

- Реальные токены, секреты, инстансы gbrain или БД – только примеры, placeholders, инструкции по созданию своих.
- Чужие приватные репозитории (внутренние реестры, операторские workspace-ы) – только публичные ссылки.
- Кастомные интеграции под конкретные продукты (готовые боты со своей бизнес-логикой) – на это у каждой команды есть свой приватный репо.

Этот репо – база. Дальше каждый ученик собирает свой стек: какие агенты ему нужны, какие skills подключить, какие интеграции дописать.
