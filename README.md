# AgentOS Skills

Skills-пак для Claude Code, используемый в интенсиве **AgentOS** ([Edge Lab](https://edgelab.su)).

После установки Claude Code получает два специализированных навыка:

- [**senior-brainstorm**](./senior-brainstorm/) — senior full-stack архитектор для EdTech-платформ. Активируется на вопросах архитектуры, выбора стека, AI/LLM интеграции.
- [**telegram-bot-builder**](./telegram-bot-builder/) — генератор Telegram-ботов на Python (aiogram 3 / python-telegram-bot v21) или TypeScript (grammY). Поддерживает SQLite, Supabase, Postgres, Yandex Cloud, Redis.

## Установка

### Через `.skill` (рекомендуется)

Одна команда — Claude Code сам скачает архив и подключит скилл:

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

После копирования перезапусти Claude Code — скиллы появятся в списке.

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

## Связанные ресурсы

- **Superpowers** — официальный фреймворк навыков для Claude Code от obra. Ставится через official Claude plugin marketplace: `/plugin install superpowers@claude-plugins-official`. Репо: [obra/superpowers](https://github.com/obra/superpowers).
- **Гайд по CLAUDE.md** — [docs/efir-1/claude-md-guide.md](https://github.com/qwwiwi/intensive-agentos/blob/main/docs/efir-1/claude-md-guide.md) из репо `intensive-agentos`.

## Лицензия

Каждый скилл наследует лицензию из своего подкаталога (см. соответствующий `LICENSE`).
