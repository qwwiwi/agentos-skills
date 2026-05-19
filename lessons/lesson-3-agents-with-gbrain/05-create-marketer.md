# 05. Создаём Marketer – контент-агент

Третий агент – Marketer. Owns публичное Instagram-присутствие: пишет reels, hooks, captions, scripts. Скрапит референсные аккаунты (например healthy-breakfast niche 100k+) через Apify / HikerAPI / Cobalt / Whisper. Держит TOV canonical snapshot и отказывается публиковать без него.

В отличие от Edith (которая всё сохраняет) – Marketer **производит** контент наружу. В отличие от Homer (который пишет код) – Marketer не лезет в production-код, не делает архитектуру.

## Что делаем

1. Запускаем `install.sh` для третьего workspace
2. Скачиваем marketer template
3. Подставляем плейсхолдеры
4. Заводим Telegram-бот через BotFather
5. Smoke-тест: просим 3 hooks на тему – Marketer выдаёт варианты, не публикует

Времени: ~10 минут.

## Шаг 1: создаём workspace

```bash
cd ~/projects/public-architecture-claude-code
bash install.sh
```

Ответы:

| Вопрос | Ответ |
|---|---|
| Agent name | `marketer` |
| Agent role | `marketer` |
| Default model | `opus` (контент требует качества) |
| Workspace path | `~/.claude-lab/marketer/.claude` (default) |
| Language | `russian` |

Получаешь workspace `~/.claude-lab/marketer/.claude/`.

## Шаг 2: скачиваем marketer template

Шаблон `marketer.md` уже содержит контракт «контент + lead-gen + public presence», без адаптации:

```bash
curl -fsSL \
  https://raw.githubusercontent.com/qwwiwi/agentos-skills/main/templates/claude-md/marketer.md \
  > ~/.claude-lab/marketer/.claude/CLAUDE.md
```

## Шаг 3: подставляем плейсхолдеры

В `marketer.md` плейсхолдеры:

```
{{Имя агента}}            – Marketer
{{my_name}}, {{agent_name}} – marketer
{{Владелец}}, {{владелец}} – Имя владельца
{{владельца}}, {{владельцу}} – падежи
{{обращение к владельцу}} – как к тебе обращаться
{{coordinator}}, {{coder}} – homer
{{crm}}                   – sales (опциональный 4-й агент, оставь как заглушку)
{{secondbrain}}           – edith
{{my_bot}}                – your_marketer_bot
{{tov_source_path}}       – путь к TOV-файлу (core/TONE_OF_VOICE.md)
{{gbrain_host}}           – your-gbrain.example.com
```

**Вариант 1 – sed с пайп-разделителем (работает на macOS и Linux):**

```bash
cd ~/.claude-lab/marketer/.claude

OWNER_NAME="Дашa"; OWNER_GEN="Даши"; OWNER_DAT="Даше"; OWNER_ADDR="босс"
GBRAIN="your-gbrain.example.com"; MY_BOT="your_marketer_bot"

sed -i.bak "s|{{Имя агента}}|Marketer|g; s|{{my_name}}|marketer|g; s|{{agent_name}}|marketer|g" CLAUDE.md
sed -i.bak "s|{{Владелец}}|${OWNER_NAME}|g; s|{{владелец}}|${OWNER_NAME}|g" CLAUDE.md
sed -i.bak "s|{{владельца}}|${OWNER_GEN}|g; s|{{владельцу}}|${OWNER_DAT}|g" CLAUDE.md
sed -i.bak "s|{{обращение к владельцу}}|${OWNER_ADDR}|g" CLAUDE.md
sed -i.bak "s|{{coordinator}}|homer|g; s|{{coder}}|homer|g" CLAUDE.md
sed -i.bak "s|{{crm}}|sales|g; s|{{secondbrain}}|edith|g" CLAUDE.md
sed -i.bak "s|{{my_bot}}|${MY_BOT}|g" CLAUDE.md
sed -i.bak "s|{{tov_source_path}}|core/TONE_OF_VOICE.md|g" CLAUDE.md
sed -i.bak "s|{{gbrain_host}}|${GBRAIN}|g" CLAUDE.md

rm -f CLAUDE.md.bak
```

**Вариант 2 – Python (надёжнее для unicode):**

```bash
python3 <<'PY'
from pathlib import Path
mapping = {
    "Имя агента": "Marketer", "my_name": "marketer", "agent_name": "marketer",
    "Владелец": "Дашa", "владелец": "Дашa",
    "владельца": "Даши", "владельцу": "Даше",
    "обращение к владельцу": "босс",
    "coordinator": "homer", "coder": "homer",
    "crm": "sales", "secondbrain": "edith",
    "my_bot": "your_marketer_bot",
    "tov_source_path": "core/TONE_OF_VOICE.md",
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
grep '{{' ~/.claude-lab/marketer/.claude/CLAUDE.md   # должно быть пусто
```

## Шаг 4: заводим Telegram-бот

Если на шаге 02 уже создал – пропусти. Если нет:

1. Открой Telegram, найди `@BotFather`
2. Отправь `/newbot`
3. Введи имя: `Marketer Growth`
4. Введи username: `your_marketer_bot` (должен заканчиваться на `bot`)
5. Получишь токен – сохрани

```bash
mkdir -p ~/.claude-lab/marketer/secrets
echo "your-marketer-bot-token" > ~/.claude-lab/marketer/secrets/telegram-bot-token
chmod 600 ~/.claude-lab/marketer/secrets/telegram-bot-token
```

**Privacy:** `@BotFather` → `/mybots` → бот → `Bot Settings` → `Group Privacy` → **Enable**. Бот в группе реагирует только на @mention / reply.

## Шаг 5: положим стартовый TOV

Marketer отказывается публиковать без TOV. Сделаем минимальную заглушку:

```bash
cat > ~/.claude-lab/marketer/.claude/core/TONE_OF_VOICE.md <<'EOF'
# TONE OF VOICE – baseline

- Краткость > объяснений
- Короткое тире (–), не em dash (—)
- Русские кавычки «»
- Без эмодзи в системных коммуникациях
- В контенте – вариативный голос по площадке (Reels короче, captions plain)
- Запрещено: канцеляризм, безличные конструкции, корпоративные клише
EOF
```

Этот файл – baseline. Marketer будет дополнять / уточнять его через time + примеры.

## Шаг 6: smoke-тест

```bash
cd ~/.claude-lab/marketer/.claude
claude
```

Спроси:

```text
Marketer, набросай 3 hooks для Reels на тему «AI-агенты для соло-фаундера».
Только варианты, не публикуй. Покажи TOV-проверку каждого.
```

Ожидаемый ответ:

- 3 варианта hooks (≤7 слов каждый)
- Для каждого: TOV-чек (короткое тире, без эмодзи, без канцеляризма)
- Уточнение: «Под какую площадку финал – Reels / Shorts / TikTok?» (Marketer спрашивает контекст ПЕРЕД публикацией)
- НЕТ публикации, никакого «опубликовал», только варианты

Если Marketer пошёл публиковать сам, без подтверждения – проверь зоны автономности в CLAUDE.md (публикация должна быть RED, требовать explicit OK).

Выйди из Claude Code.

## Troubleshooting

### Проблема: install.sh жалуется на «workspace exists»

```bash
rm -rf ~/.claude-lab/marketer
# Запусти install.sh заново
```

### Проблема: Marketer пытается писать код / архитектуру

Значит template скачался не тот или плейсхолдер `{{coder}}` не подставился. Marketer должен делегировать код через `swarm.notify(to_agent="homer", ...)`. Проверь:

```bash
grep -A 3 'production-код\|архитектур' ~/.claude-lab/marketer/.claude/CLAUDE.md
# Должна быть секция про границы роли
```

Если нет – перекачай template, повтори sed / Python.

### Проблема: остались `{{плейсхолдеры}}` после sed

`grep '{{' CLAUDE.md` покажет какие. Часто редкие падежи. Закрой руками или допиши в Python-блок.

### Проблема: Marketer публикует без TOV

Это нарушение hard rule. В шаблоне есть правило «без TOV – не публикую». Проверь, что TOV-файл лежит по пути `core/TONE_OF_VOICE.md` (шаг 5), и что в CLAUDE.md остался ссылка на этот путь. Если нет – перекачай и повтори.

### Проблема: путаются три workspace

После трёх install у тебя должны быть:

```bash
ls ~/.claude-lab/
# homer/
# edith/
# marketer/
```

Если какой-то отсутствует или лежит лишний – проверь, не залип ли install.sh на одно имя.

## Готово

Три агента стоят локально. Теперь нужна общая память – переходи к [06-setup-gbrain.md](./06-setup-gbrain.md).
