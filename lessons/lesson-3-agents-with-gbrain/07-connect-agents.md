# 07. Подключаем агентов к gbrain

gbrain поднят, токены сгенерированы. Теперь надо чтобы каждый из трёх workspace знал, как до него достучаться. Это делается через `.mcp.json` в каждом workspace плюс boot sequence в CLAUDE.md.

## Что делаем

1. В каждый workspace (`homer`, `edith`, `marketer`) кладём `.mcp.json` с четырьмя MCP-серверами gbrain
2. Подставляем свой Bearer-токен в headers
3. Перезапускаем Claude Code в каждом workspace
4. Smoke: вызываем recall – получаем 200 OK с пустыми результатами
5. Убеждаемся, что boot sequence работает (list_my_pending + task_list + heartbeat)

Времени: ~10 минут.

## Шаг 1: .mcp.json для Homer

```bash
TOKEN=$(cat ~/.claude-lab/homer/secrets/gbrain.token)
GBRAIN_URL="https://your-gbrain.example.com"

cat > ~/.claude-lab/homer/.claude/.mcp.json <<EOF
{
  "mcpServers": {
    "gbrain-memory": {
      "type": "http",
      "url": "${GBRAIN_URL}/memory/mcp",
      "headers": {
        "Authorization": "Bearer ${TOKEN}"
      }
    },
    "gbrain-recall": {
      "type": "http",
      "url": "${GBRAIN_URL}/recall/mcp",
      "headers": {
        "Authorization": "Bearer ${TOKEN}"
      }
    },
    "gbrain-swarm": {
      "type": "http",
      "url": "${GBRAIN_URL}/swarm/mcp",
      "headers": {
        "Authorization": "Bearer ${TOKEN}"
      }
    },
    "gbrain-tasks": {
      "type": "http",
      "url": "${GBRAIN_URL}/tasks/mcp",
      "headers": {
        "Authorization": "Bearer ${TOKEN}"
      }
    }
  }
}
EOF

chmod 600 ~/.claude-lab/homer/.claude/.mcp.json
```

## Шаг 2: .mcp.json для Edith

```bash
TOKEN=$(cat ~/.claude-lab/edith/secrets/gbrain.token)
GBRAIN_URL="https://your-gbrain.example.com"

cat > ~/.claude-lab/edith/.claude/.mcp.json <<EOF
{
  "mcpServers": {
    "gbrain-memory": {
      "type": "http",
      "url": "${GBRAIN_URL}/memory/mcp",
      "headers": { "Authorization": "Bearer ${TOKEN}" }
    },
    "gbrain-recall": {
      "type": "http",
      "url": "${GBRAIN_URL}/recall/mcp",
      "headers": { "Authorization": "Bearer ${TOKEN}" }
    },
    "gbrain-swarm": {
      "type": "http",
      "url": "${GBRAIN_URL}/swarm/mcp",
      "headers": { "Authorization": "Bearer ${TOKEN}" }
    },
    "gbrain-tasks": {
      "type": "http",
      "url": "${GBRAIN_URL}/tasks/mcp",
      "headers": { "Authorization": "Bearer ${TOKEN}" }
    }
  }
}
EOF

chmod 600 ~/.claude-lab/edith/.claude/.mcp.json
```

## Шаг 3: .mcp.json для Marketer

```bash
TOKEN=$(cat ~/.claude-lab/marketer/secrets/gbrain.token)
GBRAIN_URL="https://your-gbrain.example.com"

cat > ~/.claude-lab/marketer/.claude/.mcp.json <<EOF
{
  "mcpServers": {
    "gbrain-memory": {
      "type": "http",
      "url": "${GBRAIN_URL}/memory/mcp",
      "headers": { "Authorization": "Bearer ${TOKEN}" }
    },
    "gbrain-recall": {
      "type": "http",
      "url": "${GBRAIN_URL}/recall/mcp",
      "headers": { "Authorization": "Bearer ${TOKEN}" }
    },
    "gbrain-swarm": {
      "type": "http",
      "url": "${GBRAIN_URL}/swarm/mcp",
      "headers": { "Authorization": "Bearer ${TOKEN}" }
    },
    "gbrain-tasks": {
      "type": "http",
      "url": "${GBRAIN_URL}/tasks/mcp",
      "headers": { "Authorization": "Bearer ${TOKEN}" }
    }
  }
}
EOF

chmod 600 ~/.claude-lab/marketer/.claude/.mcp.json
```

## Шаг 4: проверяем синтаксис

```bash
for agent in homer edith marketer; do
  echo "=== $agent ==="
  jq . ~/.claude-lab/$agent/.claude/.mcp.json | head -3
done
```

Если `jq` парсит без ошибок – синтаксис ок. Если падает – проверь, что переменные `$TOKEN` и `$GBRAIN_URL` были выставлены перед `cat`.

## Шаг 5: добавляем boot sequence в CLAUDE.md

Каждый агент при старте должен:

1. Проверить инбокс
2. Проверить свои задачи
3. Отправить heartbeat

Открой CLAUDE.md каждого агента и убедись, что есть секция вроде:

```markdown
## Boot sequence

При старте каждой сессии:

1. \`mcp__gbrain-swarm__list_my_pending(agent="<my_name>")\` – проверить инбокс
2. \`mcp__gbrain-tasks__task_list(assignee="<my_name>", status="new")\` – новые задачи
3. \`mcp__gbrain-tasks__task_list(assignee="<my_name>", status="progress")\` – в работе
4. \`mcp__gbrain-tasks__agent_heartbeat(agent="<my_name>", status="online")\` – отметиться
```

Где `<my_name>` – `homer` / `edith` / `marketer` соответственно. В шаблонах из `agentos-skills/templates/claude-md/` эта секция уже есть, плейсхолдер `<my_name>` заменяется на этапе sed (см. шаги 03-05).

## Шаг 6: smoke-тест recall

Запусти Claude Code в одном из workspace, например, в Homer:

```bash
cd ~/.claude-lab/homer/.claude
claude
```

В диалоге попроси:

```text
Сделай вызов mcp__gbrain-recall__recall с query="hello" и limit=5.
Покажи мне сырой ответ.
```

Claude Code должен вызвать tool, получить ответ от gbrain. Ожидаемо:

```json
{
  "matches": [],
  "took_ms": 12
}
```

Пустые matches – нормально, в gbrain пока нет данных. Главное – 200 OK и валидный JSON.

Если получаешь ошибку:

- `401 Unauthorized` – токен не сходится. Проверь шаг 6 предыдущего урока (sha256 без `\n`)
- `connection refused` – gbrain упал; на VPS проверь systemctl
- `tool not found` – Claude Code не подцепил `.mcp.json`. Выйди из сессии, проверь что файл в `<workspace>/.claude/.mcp.json`, перезапусти

## Шаг 7: smoke-тест swarm.list_my_pending

В той же сессии Homer:

```text
Сделай вызов mcp__gbrain-swarm__list_my_pending с agent="homer".
```

Ожидаемо:

```json
{
  "pending": [],
  "count": 0
}
```

Пусто – нормально, никто Homer ничего не слал. Главное – 200 OK.

## Шаг 8: heartbeat от всех трёх

В каждом workspace по очереди (выйди, перейди, запусти):

```bash
cd ~/.claude-lab/homer/.claude && claude     # затем Heartbeat-вызов
cd ~/.claude-lab/edith/.claude && claude     # затем Heartbeat-вызов
cd ~/.claude-lab/marketer/.claude && claude  # затем Heartbeat-вызов
```

В каждой сессии:

```text
Сделай mcp__gbrain-tasks__agent_heartbeat(agent="<имя>", status="online").
```

После всех трёх heartbeat – проверь, что они все «видны» друг другу. В любом workspace:

```text
Сделай mcp__gbrain-tasks__agent_list. Покажи всех.
```

Ожидаемо: три записи (`homer`, `edith`, `marketer`), у всех `status: online`, `last_seen` – недавно.

## Troubleshooting

### Claude Code не видит .mcp.json

- Проверь имя файла: точно `.mcp.json` (с точкой), не `mcp.json`
- Проверь путь: `<workspace>/.claude/.mcp.json` – внутри `.claude` директории
- Проверь, что Claude Code запущен **из** этой директории: `pwd` перед `claude`

### Все три агента видят одинаковую identity

Симптом: heartbeat от Edith в `agent_list` показан как `homer`. Значит у тебя один и тот же токен в .mcp.json у разных агентов. Перепроверь – каждый workspace должен иметь свой токен из своего `secrets/gbrain.token`.

### MCP-сервер не отвечает, хотя на curl работает

- Часто из-за `type: http` vs `type: sse`. В нашем случае `type: http` – Claude Code сам определяет streamable-http
- Если используешь старую версию Claude Code (<2.1) – обнови

### Heartbeat есть, но agent_list пустой

- Проверь, что `agent_heartbeat` действительно прошёл (не 401)
- В сессии можно посмотреть последние tool calls и их статус

## Готово

Три агента подключены к gbrain, все онлайн. Переходи к [08-test-coordination.md](./08-test-coordination.md) – проверим, что они могут координироваться.
