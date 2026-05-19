# Layer 1: Workspace для агента

Репо: [`qwwiwi/public-architecture-claude-code`](https://github.com/qwwiwi/public-architecture-claude-code)

## Что это

Универсальный workspace для Claude Code-агента. Не темплейт-болванка, а готовая структура: память по слоям, hooks для self-improvement, semantic search по локальным заметкам, конфиги, скрипты для бэкапа. Запускаете `install.sh`, отвечаете на 5–7 вопросов – получаете рабочего агента.

## Что внутри

```
~/.claude-lab/{agent-name}/.claude/
├── CLAUDE.md                # identity агента
├── core/
│   ├── rules.md             # operational rules
│   ├── hot/recent.md        # последняя активность (5–15 минут)
│   ├── warm/decisions.md    # решения, контекст последних дней
│   ├── cold/MEMORY.md       # архив за пределами warm-окна
│   ├── tools/TOOLS.md       # известные инструменты, паттерны
│   └── agents/AGENTS.md     # команда (кто за что отвечает)
├── hooks/                   # PreToolUse / PostToolUse / Stop
├── scripts/                 # ротация памяти, бэкапы
└── settings.json            # конфигурация Claude Code
```

## Слойная память

Память разделена на 4 уровня, чтобы контекст-окно не забивалось мусором:

| Слой | Файл | Объём | Загружается в контекст? |
|---|---|---|---|
| HOT | `core/hot/recent.md` | последние ~10 событий | да, автоматически |
| WARM | `core/warm/decisions.md` | последние 7–14 дней | да, автоматически |
| COLD | `core/cold/MEMORY.md` | архив | нет, по запросу через grep |
| L4 | semantic index (Kuzu / LanceDB) | всё с момента установки | нет, по recall-команде |

Ротация HOT → WARM → COLD идёт по cron-у. Тяжёлые ноты не висят в контексте, лёгкие – всегда под рукой.

## Key feature: 400K working context, а не 1M

Claude Code умеет 1M context, но архитектура намеренно ограничена 400K. Почему:

- При 1M контексте модель «теряет фокус» на ранних токенах – доказано в публичных бенчмарках Anthropic.
- 400K хватает на все рабочие сценарии (3–5 средних файлов кода + рулбук + история сессии).
- Освободившиеся 600K «бюджета» уходят на хуки, semantic recall и subagents – они подгружают релевантные данные точечно, а не валят всё в контекст.

Это инверсия классического подхода «грузим всё, авось поможет». Здесь – грузим только нужное, остальное – по запросу.

## Hooks для self-improvement

В установке есть три типа хуков:

- **PreToolUse** – перехватывает опасные операции (`rm -rf`, force push, изменение секретов) и требует подтверждения.
- **PostToolUse** – ловит ошибки и автоматически пишет learning-ноту, чтобы агент не повторил ошибку.
- **Stop** – в конце сессии прогоняет компакцию памяти и обновляет hot/warm.

Хуки – обычные bash-скрипты, можно дописывать свои.

## Semantic search

Поверх локальных markdown-файлов работает индекс (Kuzu + LanceDB). Поиск идёт не по grep-у, а по смыслу: спрашиваете «что мы решили по auth?» – получаете релевантные decisions, даже если в них нет слова «auth».

## Когда использовать

| Сценарий | Подходит? |
|---|---|
| Новый агент с нуля | да, это основной use case |
| Рефакторинг существующего workspace | да, install.sh умеет merge-режим |
| Multi-agent setup (несколько агентов на одном хосте) | да, каждому свой `~/.claude-lab/{name}/.claude/` |
| Просто эксперимент со скиллами | избыточно – достаточно базового Claude Code без install.sh |

## Как установить

```bash
git clone https://github.com/qwwiwi/public-architecture-claude-code.git
cd public-architecture-claude-code
bash install.sh
```

Скрипт спросит:
- имя агента (`coder`, `marketer`, etc)
- роль (одна строка для CLAUDE.md)
- модель по умолчанию (Opus / Sonnet)
- куда ставить (default: `~/.claude-lab/{name}/.claude/`)

После установки workspace готов к работе. Запускаете `claude` из директории workspace – попадаете в свежесозданного агента.

## Где живёт workspace

```
~/.claude-lab/
├── coder/.claude/         # первый агент
├── marketer/.claude/      # второй
└── sales/.claude/         # третий
```

Каждый workspace полностью изолирован: своя память, свои хуки, своя identity. Между ними нет общего состояния – для этого нужен Layer 2 (gbrain).

## Внешняя ссылка

Подробная документация и issues: https://github.com/qwwiwi/public-architecture-claude-code
