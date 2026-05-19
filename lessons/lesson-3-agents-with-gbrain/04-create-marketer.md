# 04. Создаём marketer-агента (Маркет)

Второй агент – контент и маркетинг. Для удобства в уроке зовём его «Маркет» (`kael`). Этот агент будет писать посты, готовить сценарии, делать контент-планы.

В этом шаге также подключаем Telegram-бот к marketer'у – так удобнее: контентные задачи часто прилетают мобильно, и хочется отвечать с телефона.

## Что делаем

1. Запускаем `install.sh` для второго workspace
2. Скачиваем marketer template
3. Подставляем плейсхолдеры
4. Заводим Telegram-бот через BotFather (если ещё не сделал)
5. Smoke-тест: проверяем, что marketer знает свою специализацию

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
| Agent name | `kael` |
| Agent role | `marketer` |
| Default model | `opus` |
| Workspace path | `~/.claude-lab/kael/.claude` (default) |
| Language | `russian` |

Получаешь второй workspace `~/.claude-lab/kael/.claude/`.

## Шаг 2: скачиваем marketer template

```bash
curl -fsSL \
  https://raw.githubusercontent.com/qwwiwi/agentos-skills/main/templates/claude-md/marketer.md \
  > ~/.claude-lab/kael/.claude/CLAUDE.md
```

## Шаг 3: подставляем плейсхолдеры

В `marketer.md` 15 плейсхолдеров. Полный список (включая marketer-специфичные):

```
{{Имя агента}}, {{my_name}}, {{agent_name}}
{{Владелец}}, {{владелец}}, {{владельца}}, {{владельцу}}
{{обращение к владельцу}}
{{coordinator}}, {{coder}}, {{crm}}, {{secondbrain}}
{{my_bot}}                – username Telegram-бота без @ (your_marketer_bot)
{{tov_source_path}}       – путь к TOV-файлу (например core/TONE_OF_VOICE.md)
{{gbrain_host}}           – домен gbrain
```

**Вариант 1 – sed с пайп-разделителем (работает на macOS и Linux):**

```bash
cd ~/.claude-lab/kael/.claude

OWNER_NAME="Дашa"; OWNER_GEN="Даши"; OWNER_DAT="Даше"; OWNER_ADDR="босс"
GBRAIN="your-gbrain.example.com"; MY_BOT="your_marketer_bot"

sed -i.bak "s|{{Имя агента}}|Маркет|g; s|{{my_name}}|kael|g; s|{{agent_name}}|kael|g" CLAUDE.md
sed -i.bak "s|{{Владелец}}|${OWNER_NAME}|g; s|{{владелец}}|${OWNER_NAME}|g" CLAUDE.md
sed -i.bak "s|{{владельца}}|${OWNER_GEN}|g; s|{{владельцу}}|${OWNER_DAT}|g" CLAUDE.md
sed -i.bak "s|{{обращение к владельцу}}|${OWNER_ADDR}|g" CLAUDE.md
sed -i.bak "s|{{coordinator}}|homer|g; s|{{coder}}|homer|g" CLAUDE.md
sed -i.bak "s|{{crm}}|closer|g; s|{{secondbrain}}|secondbrain|g" CLAUDE.md
sed -i.bak "s|{{my_bot}}|${MY_BOT}|g" CLAUDE.md
sed -i.bak "s|{{tov_source_path}}|core/TONE_OF_VOICE.md|g" CLAUDE.md
sed -i.bak "s|{{gbrain_host}}|${GBRAIN}|g" CLAUDE.md

rm -f CLAUDE.md.bak
```

**Вариант 2 – Python (надёжнее для unicode, кросс-платформенный):**

```bash
python3 <<'PY'
from pathlib import Path
mapping = {
    "Имя агента": "Маркет", "my_name": "kael", "agent_name": "kael",
    "Владелец": "Дашa", "владелец": "Дашa",
    "владельца": "Даши", "владельцу": "Даше",
    "обращение к владельцу": "босс",
    "coordinator": "homer", "coder": "homer",
    "crm": "closer", "secondbrain": "secondbrain",
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
grep '{{' ~/.claude-lab/kael/.claude/CLAUDE.md   # покажет, если что-то осталось
```

Если выводит пусто – все плейсхолдеры подставлены. Если остались – закрой руками в редакторе.

## Шаг 4: заводим Telegram-бот

Если на шаге 02 уже создал – пропусти. Если нет:

1. Открой Telegram, найди `@BotFather`
2. Отправь `/newbot`
3. Введи имя: `Маркет Marketer`
4. Введи username: `your_marketer_bot` (должен заканчиваться на `bot`)
5. Получишь токен вида `123456789:AAH...` – сохрани

Сохрани токен в файл:

```bash
mkdir -p ~/.claude-lab/kael/secrets
echo "your-bot-token-here" > ~/.claude-lab/kael/secrets/telegram-bot-token
chmod 600 ~/.claude-lab/kael/secrets/telegram-bot-token
```

**Важно:** в `BotFather` пройди в `/mybots` → выбери бота → `Bot Settings` → `Group Privacy` → **включи** (Privacy ON). Иначе бот будет реагировать на каждое сообщение в группе, а не только на @mention.

## Шаг 5: добавим Telegram-конфиг в CLAUDE.md

Открой `~/.claude-lab/kael/.claude/CLAUDE.md` в редакторе. Найди секцию `## Communication` (или `## Каналы связи`) и добавь:

```markdown
## Telegram

- Bot: @your_marketer_bot
- Token: ~/.claude-lab/kael/secrets/telegram-bot-token
- Bridge: dashi-plugin-claude-code (опционально, настройка в lessons/...)
- Privacy: ON (реагируем только на @mention или reply)
```

Telegram-мост (`dashi-plugin-claude-code`) подключим отдельно после урока – сейчас токен просто лежит готовый.

## Шаг 6: smoke-тест

```bash
cd ~/.claude-lab/kael/.claude
claude
```

Спроси:

```text
Кто ты? Какая у тебя специализация? Можешь ли ты писать production-код?
```

Ожидаемый ответ:

```text
Я Маркет, marketer-агент. Пишу посты, контент-планы,
сценарии для видео, тексты лендингов. Production-код не пишу –
для этого есть coder-агент. При запросе на код – передам задачу
ему через swarm.notify.
```

Если агент знает свою роль и понимает границы – marketer готов.

Выйди из Claude Code.

## Troubleshooting

### Проблема: install.sh жалуется на «workspace exists»

Скрипт защищает от перезатирания. Если ты случайно второй раз запускаешь для того же имени:

```bash
# Удали старый workspace (ВНИМАНИЕ: только если он пустой и не нужен)
rm -rf ~/.claude-lab/kael
# Запусти install.sh заново
```

### Проблема: BotFather не присылает токен

Очень редко, но Telegram режет некоторые username'ы. Попробуй другой username (не используй `bot`, `admin`, `support` в начале).

### Проблема: marketer пытается писать код

Значит template не до конца прижился. Проверь:

```bash
grep -A 3 'Production-код' ~/.claude-lab/kael/.claude/CLAUDE.md
```

Должна быть секция про границы роли. Если нет – перекачай template, повтори sed.

### Проблема: путаются имена с шага 03

Если ты случайно запустил install для `kael` в workspace `homer` – откати:

```bash
ls ~/.claude-lab/   # должно быть две папки: homer/ и kael/
```

Если что-то не так – удали лишнее и запусти install заново.

## Готово

Маркет стоит. Переходи к [05-create-sales.md](./05-create-sales.md) – заведём sales-агента.
