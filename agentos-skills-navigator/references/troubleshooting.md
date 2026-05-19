# Troubleshooting – AgentOS Skills

Частые проблемы у учеников и быстрые фиксы. Каждый случай: симптом → диагностика → fix.

## 1. «Не нашёл template/файл в репо»

**Симптом:** ученик пишет «у вас нет файла templates/claude-md/coder.md».

**Диагностика:**

```bash
cd agentos-skills
git branch --show-current   # на какой ветке
git log --oneline -5        # последние коммиты
```

Возможно, ученик клонировал старую ветку или fork без последних коммитов.

**Fix:**

```bash
git fetch origin
git checkout main
git pull origin main
ls templates/claude-md/
```

## 2. «claude skill add не работает»

**Симптом:** `claude skill add <url>` падает с «invalid URL» или «not found».

**Диагностика:** проверь URL. Правильный формат – ссылка на `.skill` bundle через `raw.githubusercontent.com`:

```
https://github.com/qwwiwi/agentos-skills/raw/main/<skill-name>.skill
```

Не указывай ссылку на папку (`/tree/main/<skill-name>/`) – `claude skill add` ждёт именно файл `.skill`.

**Fix:**

```bash
# Правильно (через .skill bundle, если он опубликован):
claude skill add https://github.com/qwwiwi/agentos-skills/raw/main/senior-brainstorm.skill

# Неправильно:
# claude skill add https://github.com/qwwiwi/agentos-skills/tree/main/senior-brainstorm/
```

Для `agentos-skills-navigator` (`.skill` bundle пока не опубликован) используется установка из исходников:

```bash
git clone https://github.com/qwwiwi/agentos-skills.git
cp -r agentos-skills/agentos-skills-navigator ~/.claude/skills/
# Перезапусти Claude Code
```

## 3. «Placeholder `{{agent_name}}` не заменился»

**Симптом:** в установленном `CLAUDE.md` остались `{{agent_name}}`, `{{owner}}` и подобные плейсхолдеры.

**Диагностика:** `sed -i` имеет разный синтаксис на macOS и Linux:

```bash
grep -r '{{agent_name}}' ~/.claude-lab/coder/.claude/
```

**Fix – macOS (BSD sed):**

```bash
sed -i '' 's/{{agent_name}}/coder/g' ~/.claude-lab/coder/.claude/CLAUDE.md
```

**Fix – Linux (GNU sed):**

```bash
sed -i 's/{{agent_name}}/coder/g' ~/.claude-lab/coder/.claude/CLAUDE.md
```

Обрати внимание: на macOS обязательно пустой аргумент `''` после `-i`.

## 4. «Лента уроков пуста»

**Симптом:** `ls lessons/` показывает только `README.md`, не видно `lesson-3-agents-with-gbrain/`.

**Диагностика:**

```bash
pwd                         # где ты сейчас
ls -la lessons/             # видимы ли скрытые файлы
git status                  # нет ли локальных удалений
```

**Fix:** убедись, что `cd` в корень репо `agentos-skills`, а не в подпапку:

```bash
cd ~/projects/agentos-skills   # или куда ты его клонировал
ls lessons/lesson-3-agents-with-gbrain/
```

## 5. «Navigator skill не триггерится»

**Симптом:** агент не подхватывает navigator при вопросах про AgentOS.

**Диагностика:** проверь, что скилл установлен и виден:

```bash
claude skill list
```

Если в списке есть `agentos-skills-navigator` – скилл загружен. Проверь, что в твоём `CLAUDE.md` нет правила, запрещающего автоматическую активацию скиллов.

**Fix:** переустанови скилл из исходников:

```bash
claude skill remove agentos-skills-navigator
rm -rf ~/.claude/skills/agentos-skills-navigator
cd ~/projects/agentos-skills && git pull origin main
cp -r agentos-skills-navigator ~/.claude/skills/
# Перезапусти Claude Code
```

И задай вопрос с явным триггером: «расскажи про agentos skills» или «как создать agentos агента».

## 6. «MCP server connect failed»

**Симптом:** агент пишет «MCP server `gbrain-memory` connection refused» или «401 Unauthorized».

**Диагностика:**

```bash
curl -i https://mcp.<your-domain>/memory/mcp   # без Bearer ждём 401
# Должен быть 401 «Unauthorized», не connection refused
```

Если connection refused – gbrain не запущен. Если 401 – скилл подключается, но token не подходит.

**Fix:**

1. Проверь, что gbrain запущен на VPS (`docker compose ps` или `systemctl status gbrain-memory`).
2. Сверь token в `.mcp.json` агента с записью в таблице `agent_tokens` gbrain (sha256 хеш от token без `\n`).
3. Подробности – в `architecture/02-gbrain-shared-memory.md`.

## 7. «Pre-commit hook блокирует коммит»

**Симптом:** `git commit` падает с «pre-commit hook failed» или похожим.

**Диагностика:**

```bash
cat .git/hooks/pre-commit          # что за hook
ls ~/.claude-lab/shared/hooks/     # общие хуки
```

Часто причина – блок code-review или env-leak detector.

**Fix:**

1. Не используй `--no-verify` – это маскирует проблему.
2. Прочитай вывод hook'а внимательно: он покажет файл и причину блока.
3. Исправь – обычно это секрет в diff'е (`API_KEY=...`) или непрошедший review.

## 8. «Не могу запустить bash-скрипт из урока»

**Симптом:** `bash scripts/setup.sh` падает с «Permission denied».

**Диагностика:**

```bash
ls -l scripts/setup.sh
```

Если первый столбец `-rw-r--r--` – нет executable бита.

**Fix:**

```bash
chmod +x scripts/setup.sh
./scripts/setup.sh
```

Или запусти через явный bash, не полагаясь на executable:

```bash
bash scripts/setup.sh
```

## 9. «Bun not found»

**Симптом:** установка plugin падает с «bun: command not found».

**Fix:** установи Bun:

```bash
curl -fsSL https://bun.sh/install | bash
exec $SHELL   # перезагрузить PATH
bun --version
```

## 10. «Telegram-бот не получает сообщения»

**Симптом:** plugin запущен, но бот молчит на сообщения в Telegram.

**Диагностика:**

```bash
# Локально (если plugin на Mac/Linux):
curl https://api.telegram.org/bot<TOKEN>/getUpdates

# Если webhook режим:
curl https://api.telegram.org/bot<TOKEN>/getWebhookInfo
```

**Fix:** убедись, что в `config.json` указан правильный bot token, и режим (polling/webhook) согласован с тем, что зарегистрировано у Telegram. Если webhook – проверь, что URL доступен извне (TLS, открытый порт).
