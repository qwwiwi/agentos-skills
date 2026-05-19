# 03. Создаём coder-агента (Homer)

Первый агент – Homer, coder / architect / coordinator. Он же выполняет роль tech lead команды: координирует Edith (Second Brain) и Marketer (контент), владеет инфраструктурой gateway / gbrain / deploys.

## Что делаем

1. Клонируем `public-architecture-claude-code` – универсальную архитектуру с install-скриптом
2. Запускаем `install.sh`, отвечаем на 3-4 вопроса
3. Получаем готовый workspace `~/.claude-lab/homer/.claude/`
4. Заливаем туда coder CLAUDE.md template из `agentos-skills`
5. Подставляем плейсхолдеры (имя, специализация)
6. Smoke-тест: запускаем Claude Code и проверяем представление

Времени: ~10 минут.

## Шаг 1: клонируем архитектуру

```bash
cd ~/projects   # или любая папка где у тебя репо
git clone https://github.com/qwwiwi/public-architecture-claude-code.git
cd public-architecture-claude-code
```

## Шаг 2: запускаем install.sh

```bash
bash install.sh
```

Скрипт интерактивный, задаст вопросы:

| Вопрос | Ответ |
|---|---|
| Agent name | `homer` |
| Agent role | `coder` |
| Default model | `opus` (или `sonnet`, если экономишь) |
| Workspace path | `~/.claude-lab/homer/.claude` (default) |
| Language | `russian` |

После завершения скрипт создаст структуру:

```
~/.claude-lab/homer/.claude/
├── CLAUDE.md           ← пустой шаблон, заменим
├── core/
│   ├── USER.md
│   ├── rules.md
│   ├── hot/
│   ├── warm/
│   └── cold/
├── memory/
├── skills/
└── settings.json
```

## Шаг 3: скачиваем coder template

Этот репо (`agentos-skills`) содержит готовые CLAUDE.md шаблоны под разные роли. Берём coder:

```bash
curl -fsSL \
  https://raw.githubusercontent.com/qwwiwi/agentos-skills/main/templates/claude-md/coder.md \
  > ~/.claude-lab/homer/.claude/CLAUDE.md
```

Проверь, что скачалось:

```bash
head -20 ~/.claude-lab/homer/.claude/CLAUDE.md
```

Должен увидеть начало шаблона с плейсхолдерами вроде `{{Имя агента}}`, `{{владелец}}`, `{{coordinator}}`, `{{gbrain_host}}`.

## Шаг 4: подставляем плейсхолдеры

В шаблоне `coder.md` есть 11 плейсхолдеров в фигурных скобках. Полный список:

```
{{Имя агента}}            – имя на русском (Homer)
{{my_name}}               – имя на английском (homer)
{{Владелец}}              – Имя_владельца с большой
{{владелец}}              – Имя_владельца с маленькой
{{владельца}}             – родительный падеж (Андрея, Даши)
{{владельцу}}             – дательный падеж
{{владельцем}}            – творительный падеж
{{обращение к владельцу}} – как агент к тебе обращается (босс, шеф)
{{coordinator}}           – имя координатора (можно сам homer)
{{gbrain_host}}           – домен gbrain (your-gbrain.example.com)
{{языке владельца}}       – язык коммитов (русском)
```

**Вариант 1 – sed (Linux/macOS), пайп-разделитель + кросс-платформенный suffix `.bak`:**

```bash
cd ~/.claude-lab/homer/.claude

# Замени Имя_владельца и your-gbrain.example.com на свои значения
OWNER_NAME="Дашa"        # как обращаться в текстах (именительный)
OWNER_GEN="Даши"          # родительный падеж (для «у владельца Даши»)
OWNER_DAT="Даше"          # дательный
OWNER_INS="Дашей"         # творительный
OWNER_ADDR="босс"         # как агент тебя зовёт
GBRAIN="your-gbrain.example.com"

sed -i.bak "s|{{Имя агента}}|Homer|g; s|{{my_name}}|homer|g" CLAUDE.md
sed -i.bak "s|{{Владелец}}|${OWNER_NAME}|g; s|{{владелец}}|${OWNER_NAME,,}|g" CLAUDE.md 2>/dev/null || \
  sed -i.bak "s|{{Владелец}}|${OWNER_NAME}|g; s|{{владелец}}|${OWNER_NAME}|g" CLAUDE.md
sed -i.bak "s|{{владельца}}|${OWNER_GEN}|g" CLAUDE.md
sed -i.bak "s|{{владельцу}}|${OWNER_DAT}|g" CLAUDE.md
sed -i.bak "s|{{владельцем}}|${OWNER_INS}|g" CLAUDE.md
sed -i.bak "s|{{обращение к владельцу}}|${OWNER_ADDR}|g" CLAUDE.md
sed -i.bak "s|{{coordinator}}|homer|g" CLAUDE.md
sed -i.bak "s|{{gbrain_host}}|${GBRAIN}|g" CLAUDE.md
sed -i.bak "s|{{языке владельца}}|русском|g" CLAUDE.md

rm -f CLAUDE.md.bak
```

**Важно про BSD vs GNU sed:**
- На macOS BSD sed требует пустой аргумент `-i ''` (или suffix-форму `-i.bak` как выше – она работает на обоих).
- Пайп-разделитель `s|...|...|g` нужен, чтобы пробелы и кириллица в плейсхолдерах (`{{обращение к владельцу}}`) не ломали парсер.
- `${OWNER_NAME,,}` (lowercase) – bash ≥4. На старом macOS bash 3 этот синтаксис не работает; второй `sed` через `|| sed ...` это запасной fallback.

**Вариант 2 – Python (надёжнее для unicode, кросс-платформенный):**

```bash
cd ~/.claude-lab/homer/.claude

python3 <<'PY'
import re
from pathlib import Path

owner = {
    "Владелец": "Дашa",
    "владелец": "Дашa",
    "владельца": "Даши",
    "владельцу": "Даше",
    "владельцем": "Дашей",
    "обращение к владельцу": "босс",
}
agent = {
    "Имя агента": "Homer",
    "my_name": "homer",
    "coordinator": "homer",
    "gbrain_host": "your-gbrain.example.com",
    "языке владельца": "русском",
}

text = Path("CLAUDE.md").read_text()
for k, v in {**owner, **agent}.items():
    text = text.replace("{{" + k + "}}", v)
Path("CLAUDE.md").write_text(text)
print("OK")
PY
```

Если в шаблоне найдутся ещё плейсхолдеры – `grep '{{' CLAUDE.md` покажет какие. Закрой их руками.

## Шаг 5: проверяем результат

```bash
grep -c '{{' ~/.claude-lab/homer/.claude/CLAUDE.md
# Должно быть 0
```

Если 0 – все плейсхолдеры подставлены. Если больше – найди и замени:

```bash
grep '{{' ~/.claude-lab/homer/.claude/CLAUDE.md
```

## Шаг 6: smoke-тест

Запускаем Claude Code в workspace:

```bash
cd ~/.claude-lab/homer/.claude
claude
```

В диалоге спроси:

```text
Кто ты? Какая у тебя роль? На каком языке отвечаешь?
```

Ожидаемый ответ (примерно):

```text
Я Homer, coder-агент. Пишу код, чиню баги, делаю рефакторинг.
Отвечаю на русском.
```

Если ответ соответствует – Homer готов. Выходи из Claude Code (Ctrl+D или `/exit`).

## Troubleshooting

### Проблема: `install.sh` падает на `command not found`

Не хватает зависимости. Скрипт обычно сразу пишет, какой:

```text
ERROR: jq is required but not installed
```

Поставь её через системный менеджер и запусти install заново.

### Проблема: после sed остались `{{...}}` плейсхолдеры

В шаблоне могут быть нестандартные плейсхолдеры под твою специфику. Найди их и подставь руками:

```bash
grep -n '{{' ~/.claude-lab/homer/.claude/CLAUDE.md
```

Открой CLAUDE.md в редакторе и заполни вручную.

### Проблема: Claude Code говорит «не вижу CLAUDE.md»

Проверь, что ты запустил `claude` из правильной папки:

```bash
pwd
# Должно быть: /home/<user>/.claude-lab/homer/.claude
```

Если нет – `cd ~/.claude-lab/homer/.claude && claude`.

## Готово

Homer стоит. Переходи к [04-create-edith.md](./04-create-edith.md) – заведём Edith, агента Second Brain.
