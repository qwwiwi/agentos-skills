# AgentOS Skills

Skills-пак для Claude Code, используемый в интенсиве **AgentOS** ([Edge Lab](https://edgelab.su)).

После установки Claude Code получает три специализированных навыка:

- [**senior-brainstorm**](./senior-brainstorm/) – senior full-stack архитектор для EdTech-платформ. Активируется на вопросах архитектуры, выбора стека, AI/LLM интеграции.
- [**telegram-bot-builder**](./telegram-bot-builder/) – генератор Telegram-ботов на Python (aiogram 3 / python-telegram-bot v21) или TypeScript (grammY). Поддерживает SQLite, Supabase, Postgres, Yandex Cloud, Redis.
- [**agentos-skills-navigator**](./agentos-skills-navigator/) – навигация по этому репозиторию: помогает ученику найти нужный template, lesson или architecture-конспект под свой вопрос.

## Установка

### Через `.skill` (рекомендуется)

Одна команда – Claude Code сам скачает архив и подключит скилл:

```bash
claude skill add https://github.com/qwwiwi/agentos-skills/raw/main/telegram-bot-builder.skill
```

Аналогично для других скиллов по мере публикации пакетов.

### Из исходников (для просмотра/форка)

Распакованные версии скиллов лежат в этом же репо:

```bash
git clone https://github.com/qwwiwi/agentos-skills.git
cp -r agentos-skills/senior-brainstorm     ~/.claude/skills/
cp -r agentos-skills/telegram-bot-builder  ~/.claude/skills/
```

После копирования перезапусти Claude Code – скиллы появятся в списке.

## Что внутри

### senior-brainstorm

Senior-архитектор для образовательных SaaS-платформ. Помогает на этапе проектирования:

- выбор stage-aware стека (MVP / growth / scale / enterprise)
- архитектурные паттерны (модульный монолит, agent-native, MCP-интеграции)
- threat modeling и compliance (GDPR, SOC 2, PCI DSS, FERPA, COPPA)
- стратегии тестирования

Триггеры: `/senior-brainstorm`, «brainstorm», «architecture», «how to build», «stack selection», «tech choice», «design this».

### telegram-bot-builder

Универсальный сборщик Telegram-ботов. 7 reference-файлов покрывают:

- TypeScript на grammY
- Python на aiogram 3 / python-telegram-bot v21
- БД: SQLite, Supabase, Postgres, Yandex Cloud, Redis
- платежи (Telegram Payments + кастомные провайдеры)
- безопасность (rate limiting, валидация, secrets)
- деплой (VPS, Docker, serverless)
- AI-интеграция (OpenAI, Anthropic, локальные LLM)

Триггеры: «сделай телеграм бота», «telegram bot», «aiogram», «grammY», «бот на питоне», «telegram-bot-builder».

## Templates – готовые CLAUDE.md

Готовые шаблоны CLAUDE.md для четырёх типов агентов. Копируешь, заменяешь placeholders, кладёшь в workspace агента – он сразу знает кто он, какие у него правила и как ходить в общую память.

- [`templates/claude-md/coder.md`](./templates/claude-md/coder.md) – для агента-разработчика (Plan Mode, TDD, code review, верификация перед done)
- [`templates/claude-md/marketer.md`](./templates/claude-md/marketer.md) – для агента-маркетолога (контент-планы, TOV, аналитика каналов)
- [`templates/claude-md/sales.md`](./templates/claude-md/sales.md) – для агента-продажника (воронка, follow-up, объекции, CRM-логирование)
- [`templates/claude-md/inbox-monitoring.md`](./templates/claude-md/inbox-monitoring.md) – для агента-мониторинга (чтение входящих, фильтрация, эскалация)

Полная инструкция по placeholders и gbrain integration – [`templates/claude-md/README.md`](./templates/claude-md/README.md).

Визуальный гид с copy-кнопками – [`templates/3-claude-md-templates-v2.html`](./templates/3-claude-md-templates-v2.html). Открой в браузере, выбери тип агента, скопируй блок в свой CLAUDE.md.

## Lessons – боевые уроки

Готовые уроки для учеников – пошагово, с проверкой smoke-тестами. Каждый шаг даёт минимально работающий результат, который можно проверить перед переходом к следующему.

- [`lessons/lesson-3-agents-with-gbrain/`](./lessons/lesson-3-agents-with-gbrain/) – создаём трёх агентов (coder / marketer / sales) и подключаем их к общему gbrain (Second Brain).
  - Время: ~90 минут
  - Outcome: три рабочих агента, которые видят shared memory и координируются через `swarm.notify`
  - 9 шагов: overview → prerequisites → создание каждого агента → setup gbrain → connect → test → troubleshooting

## Architecture – как 5 публичных репо собираются вместе

Конспекты соседних публичных репозиториев и схема, как они работают вместе. Прочитай **до** того, как браться за урок – будет понятно, что куда подключается и зачем.

- [`architecture/README.md`](./architecture/README.md) – оглавление
- [`architecture/01-claude-code-arch.md`](./architecture/01-claude-code-arch.md) – workspace generator (`public-architecture-claude-code`)
- [`architecture/02-gbrain-shared-memory.md`](./architecture/02-gbrain-shared-memory.md) – shared memory (`public-gbrain-agentos`)
- [`architecture/03-telegram-bridge.md`](./architecture/03-telegram-bridge.md) – Telegram bridge (`dashi-plugin-claude-code`)
- [`architecture/04-claude-md-guide.md`](./architecture/04-claude-md-guide.md) – гайд по CLAUDE.md (`edgelab-claude-md`)
- [`architecture/diagram.md`](./architecture/diagram.md) – общая схема, как репо собираются вместе

## agentos-skills-navigator – навигационный skill

Подключи этот skill – и Claude Code будет помогать ученику навигироваться по этому репо: куда смотреть для уроков, какой template взять, какой архитектурный конспект прочитать первым.

```bash
# .skill bundle – TBD, пока установка из исходников:
git clone https://github.com/qwwiwi/agentos-skills.git
cp -r agentos-skills/agentos-skills-navigator ~/.claude/skills/
```

Триггеры: «agentos», «как создать агента», «templates», «уроки агентос», «agentos-skills-navigator».

## Superpowers (третий обязательный скилл интенсива)

Не пак нашего репо – ставится отдельно. **Superpowers** – это complete software development methodology для Claude Code от [Jesse Vincent (obra)](https://github.com/obra/superpowers): spec-driven workflow → план → subagent-driven development → TDD по красному/зелёному. Совпадает с философией нашего CLAUDE.md гайда (Plan Mode, Subagents, Verification Before Done).

Установка для Claude Code – на выбор:

**Через официальный Anthropic marketplace** (рекомендуется):

```bash
/plugin install superpowers@claude-plugins-official
```

**Через marketplace от автора** (Jesse Vincent, обновляется быстрее):

```bash
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
```

Полная документация и инструкции для Codex CLI, Cursor, Gemini CLI, GitHub Copilot CLI – в [obra/superpowers](https://github.com/obra/superpowers).

## Связанные ресурсы

Полная карта связанных публичных репозиториев – [`docs/repo-registry.md`](./docs/repo-registry.md). Кратко:

- [`qwwiwi/agentos-skills`](https://github.com/qwwiwi/agentos-skills) – этот репо, точка входа: skills + templates + lessons + architecture
- [`qwwiwi/public-architecture-claude-code`](https://github.com/qwwiwi/public-architecture-claude-code) – workspace generator: создаёт изолированный CLAUDE workspace на агента
- [`qwwiwi/public-gbrain-agentos`](https://github.com/qwwiwi/public-gbrain-agentos) – shared memory (Second Brain): Postgres + pgvector + MCP-серверы (memory / recall / swarm / tasks)
- [`qwwiwi/dashi-plugin-claude-code`](https://github.com/qwwiwi/dashi-plugin-claude-code) – Telegram bridge: связывает Claude Code с Telegram-ботом для общения с агентом
- [`qwwiwi/edgelab-claude-md`](https://github.com/qwwiwi/edgelab-claude-md) – расширенный гайд по CLAUDE.md (анатомия из 18 элементов)

## Лицензия

Каждый скилл наследует лицензию из своего подкаталога (см. соответствующий `LICENSE`).
