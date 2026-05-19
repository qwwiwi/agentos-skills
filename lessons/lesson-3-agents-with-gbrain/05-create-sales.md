# 05. Создаём sales-агента (Клозер)

Третий агент – продажи. Зовём его «Клозер» (`closer`). Sales близок к marketer по навыкам письма, но фокус другой: воронка, follow-up с лидами, обработка возражений, CRM-логирование.

Для sales есть отдельный template `templates/claude-md/sales.md` – с готовой воронкой, скриптами и tone-of-voice. Берём его напрямую, без адаптации marketer'а.

## Что делаем

1. Создаём третий workspace через install.sh
2. Скачиваем sales template
3. Подставляем плейсхолдеры
4. Smoke-тест: проверяем понимание воронки

Времени: ~10 минут.

## Шаг 1: создаём workspace

```bash
cd ~/projects/public-architecture-claude-code
bash install.sh
```

Ответы:

| Вопрос | Ответ |
|---|---|
| Agent name | `closer` |
| Agent role | `sales` |
| Default model | `sonnet` (sales-задачи в основном текст, opus избыточен) |
| Workspace path | `~/.claude-lab/closer/.claude` (default) |
| Language | `russian` |

## Шаг 2: скачиваем sales template

```bash
curl -fsSL \
  https://raw.githubusercontent.com/qwwiwi/agentos-skills/main/templates/claude-md/sales.md \
  > ~/.claude-lab/closer/.claude/CLAUDE.md
```

## Шаг 3: подставляем плейсхолдеры

В `sales.md` плейсхолдеры аналогичны marketer (но без `{{tov_source_path}}`, `{{agent_name}}`, `{{crm}}` – sales не пишет контент, и сам выполняет CRM-роль).

**Вариант 1 – sed с пайп-разделителем:**

```bash
cd ~/.claude-lab/closer/.claude

OWNER_NAME="Дашa"; OWNER_GEN="Даши"; OWNER_DAT="Даше"; OWNER_ADDR="босс"
GBRAIN="your-gbrain.example.com"; MY_BOT="your_sales_bot"

sed -i.bak "s|{{Имя агента}}|Клозер|g; s|{{my_name}}|closer|g" CLAUDE.md
sed -i.bak "s|{{Владелец}}|${OWNER_NAME}|g; s|{{владелец}}|${OWNER_NAME}|g" CLAUDE.md
sed -i.bak "s|{{владельца}}|${OWNER_GEN}|g; s|{{владельцу}}|${OWNER_DAT}|g" CLAUDE.md
sed -i.bak "s|{{обращение к владельцу}}|${OWNER_ADDR}|g" CLAUDE.md
sed -i.bak "s|{{coordinator}}|homer|g; s|{{coder}}|homer|g" CLAUDE.md
sed -i.bak "s|{{secondbrain}}|secondbrain|g" CLAUDE.md
sed -i.bak "s|{{my_bot}}|${MY_BOT}|g" CLAUDE.md
sed -i.bak "s|{{gbrain_host}}|${GBRAIN}|g" CLAUDE.md

rm -f CLAUDE.md.bak
```

**Вариант 2 – Python:**

```bash
python3 <<'PY'
from pathlib import Path
mapping = {
    "Имя агента": "Клозер", "my_name": "closer",
    "Владелец": "Дашa", "владелец": "Дашa",
    "владельца": "Даши", "владельцу": "Даше",
    "обращение к владельцу": "босс",
    "coordinator": "homer", "coder": "homer", "secondbrain": "secondbrain",
    "my_bot": "your_sales_bot",
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
grep '{{' ~/.claude-lab/closer/.claude/CLAUDE.md   # должно быть пусто
```

## Шаг 4: smoke-тест

```bash
cd ~/.claude-lab/closer/.claude
claude
```

Спроси:

```text
Опиши свою воронку. Какие стадии есть у лида?
Что ты делаешь после первого контакта?
```

Ожидаемый ответ:

```text
Воронка: new → contacted → qualified → proposal → won/lost.

После первого контакта:
1. Логирую разговор в gbrain (recall будет видеть будущим follow-up)
2. Перевожу лид из «new» в «contacted»
3. Создаю задачу-напоминание на follow-up через 3 дня
4. Если лид «qualified» – готовлю proposal под его контекст
```

Если агент знает воронку и логику follow-up – Клозер готов.

## Шаг 5: Telegram-бот (опционально)

Аналогично шагу 04: создай через BotFather `your_sales_bot`, сохрани токен:

```bash
mkdir -p ~/.claude-lab/closer/secrets
echo "your-sales-bot-token" > ~/.claude-lab/closer/secrets/telegram-bot-token
chmod 600 ~/.claude-lab/closer/secrets/telegram-bot-token
```

И допиши секцию `## Telegram` в его CLAUDE.md (по образцу шага 04).

## Troubleshooting

### Проблема: Клозер путается, кем он работает

Проверь, что скачал именно `sales.md`, а не `marketer.md`:

```bash
head -3 ~/.claude-lab/closer/.claude/CLAUDE.md
# Должно быть «Клозер – sales / продажник», не «маркетолог»
```

Если скачался не тот файл – перекачай:

```bash
curl -fsSL https://raw.githubusercontent.com/qwwiwi/agentos-skills/main/templates/claude-md/sales.md \
  > ~/.claude-lab/closer/.claude/CLAUDE.md
```

### Проблема: install.sh ругается на `sonnet` модель

Старые версии install.sh принимают только `opus` / `haiku`. Поставь `opus`, потом руками поменяй в `~/.claude-lab/closer/.claude/settings.json`:

```json
{
  "model": "claude-sonnet-4-6"
}
```

### Проблема: остались `{{плейсхолдеры}}` после sed

`grep '{{' CLAUDE.md` покажет какие. Часто это редкие плейсхолдеры (`{{владельцем}}` – творительный падеж и т.п.). Подставь руками или допиши их в Python-блок выше.

### Проблема: путаются три workspace

После трёх install у тебя должны быть:

```bash
ls ~/.claude-lab/
# homer/
# kael/
# closer/
```

Если какой-то отсутствует или лежит лишний – проверь, не залип ли install.sh на одно и то же имя.

## Готово

Три агента стоят локально. Теперь нужна общая память – переходи к [06-setup-gbrain.md](./06-setup-gbrain.md).
