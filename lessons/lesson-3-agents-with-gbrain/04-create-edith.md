# 04. Создаём Edith – Second Brain

Второй агент – Edith. Это **Second Brain** команды (метод Karpathy): inbox для всего, что ты кидаешь – ссылок, voice-заметок, файлов, сниппетов. Pipeline `raw → wiki → output`: сначала immutable сохранение, потом LLM-компиляция в wiki, потом артефакты на запрос.

Edith параллельно дублирует summary каждого ингеста в gbrain – команда видит, что появилось во входящем, без перегрузки shared слоя полным контентом.

Важно: Edith – НЕ контент-агент. Marketer заведём на шаге 05. Edith специализируется только на хранении и компиляции знаний.

## Что делаем

1. Запускаем `install.sh` для второго workspace
2. Создаём дерево знаний `~/.claude-lab/edith/knowledge/{raw,wiki,output}/`
3. Скачиваем inbox-monitoring template (это и есть Second Brain CLAUDE.md)
4. Подставляем плейсхолдеры
5. Заводим Telegram-бот через BotFather (если ещё не сделал)
6. Smoke-тест: кидаем Edith URL → видим запись в `raw/`

Времени: ~10 минут.

## Шаг 1: устанавливаем workspace

В уроке предполагается, что ты уже клонировал `public-architecture-claude-code` на шаге 03. Если нет – вернись и склонируй.

```bash
cd ~/projects/public-architecture-claude-code
bash install.sh
```

Отвечаешь на вопросы install.sh:

| Вопрос | Ответ |
|---|---|
| Agent name | `edith` |
| Agent role | `second-brain` |
| Default model | `opus` (компиляция wiki требует качества) |
| Workspace path | `~/.claude-lab/edith/.claude` (default) |
| Language | `russian` |

Получаешь workspace `~/.claude-lab/edith/.claude/`.

## Шаг 2: создаём дерево знаний

Edith живёт по принципу Second Brain: `raw` (immutable) → `wiki` (LLM-compiled) → `output` (генерация по запросу). Создаём структуру:

```bash
mkdir -p ~/.claude-lab/edith/knowledge/raw/assets
mkdir -p ~/.claude-lab/edith/knowledge/wiki/{sources,entities,concepts,synthesis}
mkdir -p ~/.claude-lab/edith/knowledge/output

# Стартовые файлы wiki
touch ~/.claude-lab/edith/knowledge/wiki/index.md
touch ~/.claude-lab/edith/knowledge/wiki/log.md
touch ~/.claude-lab/edith/knowledge/INDEX.md
```

Проверка:

```bash
tree ~/.claude-lab/edith/knowledge -L 2
```

Ожидаемо:

```
~/.claude-lab/edith/knowledge
├── INDEX.md
├── output
├── raw
│   └── assets
└── wiki
    ├── concepts
    ├── entities
    ├── index.md
    ├── log.md
    ├── sources
    └── synthesis
```

## Шаг 3: скачиваем inbox-monitoring template

Этот шаблон в `agentos-skills` называется `inbox-monitoring.md` (исторически), но реально это полноценный Second Brain контракт – именно то, что нужно Edith:

```bash
curl -fsSL \
  https://raw.githubusercontent.com/qwwiwi/agentos-skills/main/templates/claude-md/inbox-monitoring.md \
  > ~/.claude-lab/edith/.claude/CLAUDE.md
```

Проверка:

```bash
head -5 ~/.claude-lab/edith/.claude/CLAUDE.md
# Должен начинаться с «# {{Имя агента}} – Second Brain / inbox / мониторинг»
```

## Шаг 4: подставляем плейсхолдеры

В `inbox-monitoring.md` плейсхолдеры под Second Brain-specific роль:

```
{{Имя агента}}            – Edith
{{my_name}}               – edith
{{Владелец}}              – Имя владельца (именительный)
{{владелец}}              – с маленькой
{{владельца}}             – родительный
{{владельцу}}             – дательный
{{обращение к владельцу}} – как к тебе обращаться
{{coordinator}}           – homer
{{my_bot}}                – your_edith_bot
{{целевой группы}}        – название группового чата (если не нужен мониторинг – оставь «—»)
{{store_root}}            – путь к дереву знаний (~/.claude-lab/edith/knowledge)
{{long-term store}}       – «локальный store + gbrain external_note (dual-write)»
{{my_namespace}}          – edith
{{support-skill}}         – course-support (или пусто)
{{gbrain_host}}           – your-gbrain.example.com
```

**Python-вариант (рекомендую – unicode стабильнее):**

```bash
cd ~/.claude-lab/edith/.claude

python3 <<'PY'
from pathlib import Path
mapping = {
    "Имя агента": "Edith", "my_name": "edith",
    "Владелец": "Дашa", "владелец": "Дашa",
    "владельца": "Даши", "владельцу": "Даше",
    "обращение к владельцу": "босс",
    "coordinator": "homer",
    "my_bot": "your_edith_bot",
    "целевой группы": "—",
    "store_root": "~/.claude-lab/edith/knowledge",
    "long-term store": "локальный store + gbrain external_note (dual-write)",
    "my_namespace": "edith",
    "support-skill": "course-support",
    "gbrain_host": "your-gbrain.example.com",
}
text = Path("CLAUDE.md").read_text()
for k, v in mapping.items():
    text = text.replace("{{" + k + "}}", v)
Path("CLAUDE.md").write_text(text)
print("OK")
PY
```

Проверка:

```bash
grep '{{' ~/.claude-lab/edith/.claude/CLAUDE.md   # должно быть пусто
```

Если что-то осталось – закрой руками.

## Шаг 5: заводим Telegram-бот

Если на шаге 02 уже создал `your_edith_bot` – пропусти. Если нет:

1. Открой Telegram, найди `@BotFather`
2. Отправь `/newbot`
3. Введи имя: `Edith Second Brain`
4. Введи username: `your_edith_bot` (должен заканчиваться на `bot`)
5. Получишь токен вида `123456789:AAH...` – сохрани

Сохрани токен в файл:

```bash
mkdir -p ~/.claude-lab/edith/secrets
echo "your-edith-bot-token" > ~/.claude-lab/edith/secrets/telegram-bot-token
chmod 600 ~/.claude-lab/edith/secrets/telegram-bot-token
```

**Privacy:** `@BotFather` → `/mybots` → твой бот → `Bot Settings` → `Group Privacy` → **Enable** (включить). Без этого бот реагирует на каждое сообщение в группе, а не только на @mention / reply.

## Шаг 6: smoke-тест

```bash
cd ~/.claude-lab/edith/.claude
claude
```

Спроси (как будто кидаешь ссылку на сохранение):

```text
Edith, прими этот URL: https://example.com/article-about-llm-agents

Покажи мне:
1) Что ты сделала (extract → save → dual-write)
2) Путь к записи в raw/
3) Сводку из 2-3 предложений
```

Ожидаемый ход действий:

1. Edith вызывает скилл (markdown-new / web-fetch) для извлечения контента
2. Создаёт `~/.claude-lab/edith/knowledge/raw/2026-05-19-<slug>.md` с YAML frontmatter
3. Параллельно зовёт `mcp__gbrain-memory__create_external_note(title, body, source_url, tags)` – summary 1–3 абзаца
4. Отвечает одной строкой: `Сохранил. article: <название> – <инсайт>. Источник сохранён.`

Если она пишет ответ текстом, но не делает tool-call в raw/ и в gbrain – значит SAVE workflow не сработал. Проверь, что `gbrain` MCP ещё не подключён (шаги 06-07 ниже) – тогда `create_external_note` не выполнится; запись только в raw/ это нормально для текущего шага, dual-write поднимется после подключения gbrain.

Проверка локального raw/:

```bash
ls -la ~/.claude-lab/edith/knowledge/raw/
# Ожидаем файл вида 2026-05-19-<slug>.md
```

Если файл есть и в нём YAML frontmatter + контент – Second Brain workflow работает.

Выйди из Claude Code.

## Troubleshooting

### Проблема: install.sh жалуется на «workspace exists»

Скрипт защищает от перезатирания. Если ты случайно второй раз запускаешь для того же имени:

```bash
# Удали старый workspace (ВНИМАНИЕ: только если он пустой и не нужен)
rm -rf ~/.claude-lab/edith
# Запусти install.sh заново
```

### Проблема: Edith пытается «улучшить» / «переписать» raw

Это нарушение золотого правила Karpathy Second Brain: `raw/` immutable. Проверь, что в CLAUDE.md осталась секция «10 правил wiki» с правилом «НИКОГДА не модифицировать файлы в raw/». Если нет – перекачай template, повтори sed.

### Проблема: остались `{{плейсхолдеры}}` после Python

`grep '{{' CLAUDE.md` покажет какие. Часто это редкие (`{{владельцем}}` – творительный падеж или `{{целевой группы}}`). Подставь руками или допиши их в Python-блок выше.

### Проблема: Edith «маркетит» – пишет посты, captions

Значит template скачался не тот (`marketer.md` вместо `inbox-monitoring.md`). Перекачай:

```bash
curl -fsSL https://raw.githubusercontent.com/qwwiwi/agentos-skills/main/templates/claude-md/inbox-monitoring.md \
  > ~/.claude-lab/edith/.claude/CLAUDE.md
```

И повтори шаг 4 (подстановка плейсхолдеров).

## Готово

Edith стоит. Дерево знаний создано, Second Brain workflow работает локально. Полный dual-write в gbrain активируется после шагов 06-07.

Переходи к [05-create-marketer.md](./05-create-marketer.md) – заведём Marketer.
