# Repo Map – agentos-skills

Карта всех файлов репозитория `qwwiwi/agentos-skills`. Используй, когда ученик спрашивает «а где у вас файл X».

## Корневой уровень

```
agentos-skills/
├── README.md                       Описание репо, TL;DR, ссылки на секции
├── ARCHITECTURE.md                 Как 5 публичных репо собираются вместе
├── LICENSE                         MIT, на весь репозиторий
```

## Скиллы (Claude Code skills)

```
├── senior-brainstorm/              Архитектура образовательных SaaS
│   ├── SKILL.md                    Фронтматтер + body, decision frameworks
│   ├── references/                 13 reference-файлов по доменам
│   ├── templates/                  ADR, threat-model, options-matrix
│   └── LICENSE
├── senior-brainstorm.skill         Bundle-архив для claude skill add
│
├── telegram-bot-builder/           Production telegram-боты
│   ├── SKILL.md                    Stack selection, Python + TS
│   ├── references/                 7 reference-файлов (aiogram, grammy, и т.д.)
│   └── LICENSE
├── telegram-bot-builder.skill      Bundle-архив
│
├── agentos-skills-navigator/       Навигатор по этому репо (ты сейчас здесь)
│   ├── SKILL.md
│   ├── references/
│   │   ├── repo-map.md             Этот файл
│   │   ├── install-paths.md        Пути установки всех компонентов
│   │   └── troubleshooting.md      Частые ошибки + фиксы
│   └── LICENSE
```

## Templates (готовые CLAUDE.md)

```
├── templates/
│   ├── README.md                   Зачем шаблоны, как использовать, placeholders
│   ├── claude-md/
│   │   ├── README.md               Gbrain integration guide для всех шаблонов
│   │   ├── coder.md                Шаблон для агента-кодера
│   │   ├── marketer.md             Шаблон для маркетолога
│   │   ├── sales.md                Шаблон для sales-агента (воронка, follow-up, CRM)
│   │   └── inbox-monitoring.md     Шаблон для агента мониторинга чатов
│   └── 3-claude-md-templates-v2.html  Визуальный гид (HTML, ~67KB)
```

## Lessons (пошаговые уроки)

```
├── lessons/
│   ├── README.md                   Карта всех уроков
│   └── lesson-3-agents-with-gbrain/
│       ├── README.md               Overview + outcomes
│       ├── 01-overview.md          Зачем 3 агента, какие роли
│       ├── 02-prerequisites.md     Что установить заранее
│       ├── 03-create-coder.md      Homer (coder): шаблон, workspace, старт
│       ├── 04-create-edith.md      Edith (Second Brain): inbox + raw/wiki/output
│       ├── 05-create-marketer.md   Marketer: шаблон, workspace, TOV
│       ├── 06-setup-gbrain.md      Self-host gbrain, токены, .mcp.json
│       ├── 07-connect-agents.md    Подключаем все 3 агента к gbrain
│       ├── 08-test-coordination.md Smoke swarm.notify между агентами
│       └── 09-troubleshooting.md   Распространённые ошибки урока
```

## Architecture (конспекты связанных репо)

```
├── architecture/
│   ├── README.md                   Как 5 публичных репо собираются вместе
│   ├── 01-claude-code-arch.md      Конспект public-architecture-claude-code
│   ├── 02-gbrain-shared-memory.md  Конспект public-gbrain-agentos
│   ├── 03-telegram-bridge.md       Конспект plugin для Claude Code + gateway
│   ├── 04-claude-md-guide.md       Конспект edgelab-claude-md + TL;DR
│   └── diagram.md                  ASCII-диаграмма потоков данных
```

## Docs (общая документация)

```
└── docs/
    ├── repo-registry.md            Public-safe карта связанных репо
    ├── prerequisites.md            Системные требования и установка инструментов
    └── faq.md                      Частые вопросы от учеников
```

## Как использовать эту карту

1. Ученик задал вопрос – определи класс (template / lesson / architecture / install / troubleshooting).
2. Открой эту карту, найди соответствующую папку.
3. Дай ученику конкретный путь: «открой `lessons/lesson-3-agents-with-gbrain/06-setup-gbrain.md`».
4. Не пересказывай содержимое – пусть прочитает сам.
