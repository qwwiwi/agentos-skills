# AgentOS Skills

> Публичные ресурсы для построения собственного агентского роя из Claude Code агентов. Используется в интенсиве **AgentOS** ([Edge Lab](https://edgelab.su)).

Этот репо – точка входа. Внутри: архитектура роя, шаблоны CLAUDE.md, боевой урок создания трёх агентов, навигационный skill, и три специализированных skill-а из боевой практики.

**Идея простая.** Ученик кидает своему Claude Code агенту один промпт «подними мне команду из трёх агентов» + ссылку на этот репо. Дальше агент сам читает `AGENTIC-SWARM.md`, идёт в `lessons/lesson-3-agents-with-gbrain/`, подтягивает шаблоны из `templates/`, подключается к соседним публичным репо по карте `architecture/` – и через ~90 минут у ученика рабочий рой. Один промпт. Один репо. Никаких догадок.

## Что такое агентский рой

Рой – команда из трёх Claude Code агентов, которые живут на одном VPS, делят общую память (`gbrain`) и общий inbound-канал (Telegram). Каждый агент – отдельный процесс Claude Code CLI с собственным workspace, собственным Telegram-ботом и собственным Bearer-токеном к общему мозгу. Между собой агенты разговаривают не через Telegram, а через shared event bus (`gbrain-swarm`) и shared semantic store (`gbrain-memory` + `gbrain-recall`).

Это не Kubernetes-кластер и не «AutoGen на 50 ролей». Это минимально полезная команда для одного оператора: build, remember, grow. Три роли, одна машина, boring infrastructure (systemd + Postgres + Python).

```
        Telegram (один бот на агента)
                  │
                  ▼
        Telegram Gateway (systemd, :9090)
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
    Homer      Edith     Marketer
   (coder)   (brain)    (growth)
       │          │          │
       └──────────┼──────────┘
                  ▼
         gbrain (Postgres + pgvector)
        ports 8766 / 8767 / 8768
```

Три агента:

- **Homer** – coder/architect/coordinator. Пишет код, проводит ревью, владеет инфраструктурой (gateway, gbrain, deploys).
- **Edith** – Second Brain. Ловит каждую ссылку/voice/файл от оператора, гонит через pipeline `raw → wiki → output`, дублирует summary в gbrain.
- **Marketer** – content/lead-gen. Reels, hooks, captions, TOV-стандарт, скрейп референсов.

**Полный teaching brief: [AGENTIC-SWARM.md](./AGENTIC-SWARM.md)** – 335 строк архитектуры с ASCII-диаграммами, таблицами, разбором каждого компонента (workspace anatomy, memory layers, MCP-серверы, auth model, gateway streaming, inter-agent communication, security zones, end-to-end turn). Это главный документ репо – читать первым.

## Что внутри репо

- **`AGENTIC-SWARM.md`** – полная архитектура агентского роя. Главный документ.
- **`templates/claude-md/`** – 4 шаблона CLAUDE.md: `coder.md` (Homer), `inbox-monitoring.md` (Edith – Second Brain), `marketer.md` (Marketer), `sales.md` (опциональный 4-й агент).
- **`lessons/lesson-3-agents-with-gbrain/`** – боевой урок: 9 шагов, ~90 минут, создаём Homer/Edith/Marketer и подключаем их к gbrain.
- **`architecture/`** – конспекты соседних публичных репо (`public-architecture-claude-code`, `public-gbrain-agentos`, `dashi-plugin-claude-code`, `edgelab-claude-md`) + общая диаграмма.
- **`agentos-skills-navigator/`** – навигационный skill для этого репо. Claude Code подсказывает ученику, какой файл открыть под вопрос.
- **`docs/`** – `repo-registry.md`, `prerequisites.md`, `faq.md`, `escalation.md` (что делать когда сломалось).
- **`senior-brainstorm/`** + **`telegram-bot-builder/`** – два прикладных skill-а из боевой практики. Senior-brainstorm используется Homer-ом для архитектурных решений. Telegram-bot-builder – для сборки Telegram-ботов (на случай если рою нужны user-facing боты помимо inbound от оператора).

## Установка

### Если ты хочешь поднять свой рой

Порядок такой:

```bash
# 1. Клонируй этот репо – архитектура и шаблоны
git clone https://github.com/qwwiwi/agentos-skills.git

# 2. Прочитай главный документ (~15 мин)
less agentos-skills/AGENTIC-SWARM.md

# 3. Пройди боевой урок (~90 мин)
open agentos-skills/lessons/lesson-3-agents-with-gbrain/README.md
```

По ходу урока ты подключишь четыре соседних публичных репо (см. секцию «Связанные публичные репо» ниже): workspace generator, gbrain (общая память), Telegram gateway, гайд по CLAUDE.md.

### Если нужны только skill-ы из этого репо

Skill в Claude Code ставится копированием папки в `~/.claude/skills/` — отдельной команды `claude skill add <url>` в Claude Code НЕТ:

```bash
git clone https://github.com/qwwiwi/agentos-skills.git
cp -r agentos-skills/senior-brainstorm         ~/.claude/skills/
cp -r agentos-skills/telegram-bot-builder      ~/.claude/skills/
cp -r agentos-skills/agentos-skills-navigator  ~/.claude/skills/
```

После копирования перезапусти Claude Code – skill-ы появятся в списке.

## Обязательные skill-ы для каждого агента

### Superpowers

**Superpowers** от Anthropic (и upstream от [obra](https://github.com/obra/superpowers)) – обязательный skill для каждого агента в рое. Без него агент не следует методологии: spec → план → TDD → review → verification. Это не косметика, это инженерная дисциплина в виде skill-файлов, которые загружаются в каждый prompt – модель буквально не может пропустить чеклист.

Установка (две опции):

```bash
# Official Anthropic marketplace (рекомендуется)
/plugin install superpowers@claude-plugins-official

# Upstream от автора Jesse Vincent (обновляется быстрее)
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
```

Полная документация: [obra/superpowers](https://github.com/obra/superpowers).

### Skill-ы из этого репо

Помимо Superpowers, агенты в рое используют:

- `senior-brainstorm` – обязателен для Homer (архитектурные решения, выбор стека, threat modeling).
- `telegram-bot-builder` – обязателен для Homer и Marketer, если рой собирает Telegram-ботов для конечных пользователей (не только inbound от оператора).
- `agentos-skills-navigator` – обязателен для всех агентов: помогает ученику и самим агентам ориентироваться в этом репо.

## Создание собственных skill-ов

Если рою нужен новый skill (специфичный workflow, кастомная методика, домен-знание для одного из агентов) – собираем его не вручную, а через стандартный workflow.

### skill-creator – фабрика новых skill-ов

[`skill-creator`](https://github.com/anthropics/skills/tree/main/skills/skill-creator) – официальный skill от Anthropic для создания других skill-ов. Загружает в контекст методологию: определяем триггеры, пишем SKILL.md frontmatter, бьём содержимое на progressive disclosure (метаданные → SKILL.md → bundled resources), готовим test cases, прогоняем eval, итерируем. Это не «напиши markdown руками» – это полноценный workflow с проверкой качества triggering и output-а.

Установка:

```bash
git clone https://github.com/anthropics/skills.git /tmp/anthropic-skills
cp -r /tmp/anthropic-skills/skills/skill-creator ~/.claude/skills/
# Перезапусти Claude Code
```

Использование:

```
/skill-creator
> «Создай skill для <твой workflow>. Триггеры: <фразы>. Выход: <формат>.»
```

Skill-creator проведёт через 4 шага: capture intent → interview → write SKILL.md → run evals.

### skills.sh – каталог готовых skill-ов

[`skills.sh`](https://www.skills.sh/) – каталог skill-ов от сообщества (включая [obra/superpowers](https://github.com/obra/superpowers) – тот же автор). Перед тем как писать новый skill через skill-creator – **проверь catalog**. Велик шанс, что нужная функциональность уже упакована и протестирована.

Принцип: **search-first**. Поток такой:

1. Нужна новая способность для агента?
2. Загляни на [skills.sh](https://www.skills.sh/) – есть готовый?
3. Если да – установи: с skills.sh через `npx skills add <owner/repo>`, плагин через `/plugin install <name>`, либо скопируй папку скилла в `~/.claude/skills/`.
4. Если нет – запусти `skill-creator` и сделай свой. Когда отполируешь – опубликуй обратно в catalog, чтобы кто-то ещё переиспользовал.

Так рой накапливает collective knowledge без дублирования.

## Связанные публичные репо

Этот репо – одна из пяти публичных частей AgentOS. Чтобы поднять рой, ты пройдёшь через все:

| Репо | Для чего | Когда подключить |
|------|----------|------------------|
| [`public-architecture-claude-code`](https://github.com/qwwiwi/public-architecture-claude-code) | Generator workspace для нового агента (CLAUDE.md, hooks, layered memory) | Шаги 03/04/05 урока (создание Homer/Edith/Marketer) |
| [`public-gbrain-agentos`](https://github.com/qwwiwi/public-gbrain-agentos) | Self-host shared brain (Postgres + pgvector + 3 MCP-сервера) | Шаг 06 урока (когда поднимаешь gbrain на VPS) |
| [`jarvis-telegram-gateway`](https://github.com/qwwiwi/jarvis-telegram-gateway) | Telegram gateway (Python systemd unit) – **актуален до 2026-06-15** | Шаг 07 урока (подключение Telegram) |
| [`dashi-plugin-claude-code`](https://github.com/qwwiwi/dashi-plugin-claude-code) | Замена gateway-у после 2026-06-15 (channel pattern, Bun + TS) | Шаг 07 урока после deadline |
| [`edgelab-claude-md`](https://github.com/qwwiwi/edgelab-claude-md) | Руководство по заполнению CLAUDE.md (анатомия из 18 элементов) | Перед заполнением placeholders в `templates/claude-md/` |

Полная карта со ссылками и описаниями – [`docs/repo-registry.md`](./docs/repo-registry.md).

## Эскалация: что делать если что-то сломалось

Когда рой работает – всё прозрачно. Когда ломается – ученику нужен быстрый switch на правильный runbook:

- Telegram gateway не отвечает на сообщения → см. [`docs/escalation.md#gateway`](./docs/escalation.md)
- gbrain MCP возвращает 401 / timeout → см. [`docs/escalation.md#gbrain`](./docs/escalation.md)
- Агент завис / не видит входящие сообщения → см. [`docs/escalation.md#agent`](./docs/escalation.md)
- Нужна команда диагностики «всё ли живо» → см. [`docs/escalation.md`](./docs/escalation.md)

Homer (агент-координатор) обучен читать `docs/escalation.md` при любых жалобах на инфраструктуру, до того как лезть в код.

## Лицензия

Каждый skill наследует лицензию из своего подкаталога (см. соответствующий `LICENSE`).
