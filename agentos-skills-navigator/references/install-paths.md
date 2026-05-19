# Install Paths – AgentOS

Пути установки и команды для каждого компонента AgentOS-стека. Используй, когда ученик спрашивает «куда положить X» или «какой командой ставить Y».

## Обзор 4 компонентов

| Компонент | Что это | Где живёт |
|---|---|---|
| `public-architecture-claude-code` | Архитектура workspace + install.sh | На машине ученика, в его `~/.claude-lab/` или аналогичной директории |
| `public-gbrain-agentos` | Общая память агентов (MCP server) | На VPS ученика (Hetzner / DigitalOcean / Timeweb) |
| `dashi-plugin-claude-code` | Plugin для Claude Code (Telegram bridge) | Локально на машине ученика или на VPS, рядом с агентом |
| `qwwiwi/agentos-skills` | Этот репо со скиллами и шаблонами | Любая удобная директория для git clone |

## Superpowers — mandatory для каждого агента

Перед использованием любого агента из этого репо, установи Superpowers:

`/plugin install superpowers@claude-plugins-official`

Или через obra:
`/plugin marketplace add obra/superpowers-marketplace && /plugin install superpowers@superpowers-marketplace`

Все skill-ы из этого репо (senior-brainstorm, telegram-bot-builder, agentos-skills-navigator) предполагают, что Superpowers уже установлен и активен.

## 1. public-architecture-claude-code (workspace архитектура)

**Что внутри:** install.sh, шаблон workspace, hooks для Claude Code.

**Установка:**

```bash
git clone https://github.com/qwwiwi/public-architecture-claude-code.git
cd public-architecture-claude-code
bash install.sh
```

**Куда ставится workspace одного агента:**

```
~/.claude-lab/{agent_name}/.claude/
├── CLAUDE.md                  # Идентичность агента (берётся из templates/claude-md/)
├── settings.json              # Конфиг harness
├── core/                      # Память агента
└── hooks/                     # Hook-скрипты
```

`{agent_name}` подставляется при установке (например, `coder`, `marketer`, `sales`).

**Hooks:** дополнительные общие хуки – в `~/.claude-lab/shared/hooks/`.

## 2. public-gbrain-agentos (общая память)

**Что внутри:** 4 MCP server'а (memory, recall, swarm, tasks) + миграции Postgres + HMAC auth.

**Установка через AGENT.md (рекомендуется):**

Самый простой путь – передать установку Claude Code через `AGENT.md`:

```bash
# На VPS:
cd /opt
git clone https://github.com/qwwiwi/public-gbrain-agentos.git
cd public-gbrain-agentos
claude   # в сессии: «Прочитай AGENT.md и проведи меня через установку»
```

Claude Code задаст 8–12 вопросов (домен, embedding provider, agent identities, scopes), сам поднимет Postgres, накатит миграции, сгенерирует Caddy-конфиг, запросит TLS-сертификат, поднимет 4 systemd-сервиса.

**Ручная установка – два режима:**

### Docker Compose (быстрый старт, без systemd)

```bash
git clone https://github.com/qwwiwi/public-gbrain-agentos.git
cd public-gbrain-agentos
cp .env.example .env
# Отредактируй .env: токены, домен, БД
docker compose up -d
```

Подробнее – в README `public-gbrain-agentos`.

### systemd на bare-metal VPS

```bash
# На VPS:
cd /opt
git clone https://github.com/qwwiwi/public-gbrain-agentos.git
cd public-gbrain-agentos
bash scripts/install-vps.sh
```

Создаёт unit-файлы (имена и путь установки могут отличаться в текущей версии install-vps.sh – точные значения см. в выводе скрипта): `gbrain-memory.service`, `gbrain-recall.service`, `gbrain-swarm.service`, `gbrain-tasks.service`, `gbrain-swarm-worker.service`. По умолчанию install-vps.sh кладёт файлы в `/opt/gbrain/` или предлагает выбрать свой путь.

Локальная разработка без systemd – `scripts/install-local.sh`.

**Доменная схема (если хочешь TLS):**

- Поставь Caddy перед MCP-серверами.
- DNS A-record `mcp.{your-domain}` → IP VPS.
- Caddy получит Let's Encrypt cert автоматически.

Полный гайд – README `public-gbrain-agentos`.

## 3. dashi-plugin-claude-code (Telegram bridge)

**Что внутри:** Bun-приложение, мост между Telegram-ботом и Claude Code workspace.

**Установка – локально на Mac/Linux:**

```bash
git clone https://github.com/qwwiwi/dashi-plugin-claude-code.git
cd dashi-plugin-claude-code
bun install
cp config.example.json config.json
# Отредактируй config.json: bot tokens, paths to agent workspaces
```

**Запуск:**

```bash
bun run start
```

Или через `launchctl` (macOS) / `systemd` (Linux) для автозапуска – см. README plugin.

**На VPS (если агент живёт удалённо):**

Аналогично, через `systemd` unit. Положи репо в `/opt/dashi-plugin-claude-code/`, юнит – в `/etc/systemd/system/dashi-plugin.service`.

## 4. agentos-skills (этот репо)

**Установка – просто клонировать:**

```bash
git clone https://github.com/qwwiwi/agentos-skills.git
cd agentos-skills
```

**Подключение скиллов в Claude Code:**

`senior-brainstorm` и `telegram-bot-builder` идут как `.skill` bundle, ставятся одной командой:

```bash
claude skill add https://github.com/qwwiwi/agentos-skills/raw/main/senior-brainstorm.skill
claude skill add https://github.com/qwwiwi/agentos-skills/raw/main/telegram-bot-builder.skill
```

`agentos-skills-navigator` пока ставится из исходников (`.skill` bundle TBD):

```bash
git clone https://github.com/qwwiwi/agentos-skills.git
cp -r agentos-skills/agentos-skills-navigator ~/.claude/skills/
# Перезапусти Claude Code
```

## Что куда положить (cheat-sheet)

| Что | Куда |
|---|---|
| `CLAUDE.md` для агента | `~/.claude-lab/{agent}/.claude/CLAUDE.md` |
| `.mcp.json` для агента | `~/.claude-lab/{agent}/.claude/.mcp.json` |
| Общие hooks | `~/.claude-lab/shared/hooks/` |
| Plugin config (bot tokens) | директория `dashi-plugin-claude-code/config.json` |
| gbrain secrets / tokens | На VPS в `/etc/gbrain/secrets.env` (systemd-режим, по умолчанию) или `.env` (Docker Compose) |
| Скиллы Claude Code | Управляются командой `claude skill add` (харнесс кладёт сам) |
