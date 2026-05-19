---
name: agentos-skills-navigator
description: Navigate the AgentOS skills repository – find templates, lessons, architecture refs, and skill guides. Use when a student of AgentOS intensive asks about agent setup, CLAUDE.md templates, gbrain integration, telegram bridge, or how to create a coder/marketer/sales agent. Also use when user mentions «agentos», «agentos skills», «как создать агента», «как подключить gbrain», «templates», «уроки агентос», «архитектура агентов», «3 агента».
---

# AgentOS Skills Navigator

Скилл-проводник по репозиторию `qwwiwi/agentos-skills`. Помогает ученику интенсива AgentOS быстро найти нужный шаблон, урок или архитектурный конспект – без долгих блужданий по дереву файлов.

## Что внутри репозитория

Карта папок верхнего уровня:

- `AGENTIC-SWARM.md` – **canonical teaching brief** репо: зачем нужен рой, какие роли, как делится память. Открывай первым, если ученик спрашивает «что вообще такое AgentOS».
- `senior-brainstorm/` – архитектурный скилл для образовательных SaaS.
- `telegram-bot-builder/` – скилл по сборке production telegram-ботов.
- `agentos-skills-navigator/` – этот скилл, навигация по репо.
- `templates/` – готовые шаблоны `CLAUDE.md` для разных ролей (coder, marketer, sales, inbox-monitoring) + визуальный гид HTML.
- `lessons/` – пошаговые уроки. Главный: `lesson-3-agents-with-gbrain/` (3 агента + общий gbrain за 90 минут).
- `architecture/` – конспекты 4 связанных публичных репозиториев, ASCII-диаграмма потоков данных.
- `docs/` – реестр связанных репо, prerequisites, FAQ.
- `README.md`, `ARCHITECTURE.md`, `LICENSE` – корневые файлы.

## Когда использовать этот skill

Подключай меня, когда ученик:

- Спрашивает про AgentOS, agentos skills, набор скиллов для интенсива.
- Хочет создать агента-кодера, маркетолога или sales – не знает, какой шаблон взять.
- Не понимает, как подключить gbrain (общая память агентов) к workspace.
- Ищет шаблон `CLAUDE.md` под конкретную роль.
- Хочет понять архитектуру: как 5 публичных репо собираются вместе.
- Не понимает, что такое CLAUDE.md, MCP, `.mcp.json`, plugin для Claude Code.
- Спрашивает «где у вас лежит X», «как поставить Y», «что делать если Z не работает».

НЕ подключай меня, если задача:

- Писать прод-код агента (это работа самого Claude Code, без навигатора).
- Деплоить gbrain на VPS (это `public-gbrain-agentos`, читай его README).
- Чинить production-инцидент (это другой контур).

## Как навигировать

### Шаг 1 – понять класс вопроса

| Класс вопроса | Куда вести |
|---|---|
| «дай готовый CLAUDE.md» | `templates/claude-md/` |
| «покажи пошагово как поднять 3 агента» | `lessons/lesson-3-agents-with-gbrain/` |
| «как работает архитектура» | `architecture/` + `ARCHITECTURE.md` |
| «где у вас файл X» | `references/repo-map.md` |
| «куда что устанавливается» | `references/install-paths.md` |
| «у меня ошибка Y» | `references/troubleshooting.md` |
| «какие репо связаны с этим» | `docs/repo-registry.md` |
| «хочу создать новый skill / расширить агента способностью» | секция `README.md#создание-собственных-skill-ов` + [skill-creator](https://github.com/anthropics/skills/tree/main/skills/skill-creator) + [skills.sh](https://www.skills.sh/) каталог |

### Шаг 2 – открыть нужный reference

У меня три reference-файла:

- `references/repo-map.md` – полная карта файлов репозитория с одной строкой описания.
- `references/install-paths.md` – пути установки для каждого компонента (workspace, plugin, gbrain).
- `references/troubleshooting.md` – 8-10 частых ошибок с диагностикой и фиксом.

Открывай ровно тот, который нужен сейчас – не загружай всё сразу.

### Шаг 3 – дать конкретный ответ

Не пересказывай содержимое файла – дай ученику путь, имя файла, фрагмент команды. Цель – чтобы он сам открыл нужное место и продолжил.

## Триггеры с примерами

| Запрос ученика | Куда вести |
|---|---|
| «как создать coder-агента» | `templates/claude-md/coder.md` + `lessons/lesson-3-agents-with-gbrain/03-create-coder.md` |
| «нужен Second Brain / inbox-агент» | `templates/claude-md/inbox-monitoring.md` + `lessons/lesson-3-agents-with-gbrain/04-create-edith.md` |
| «нужен маркетолог» | `templates/claude-md/marketer.md` + `lessons/lesson-3-agents-with-gbrain/05-create-marketer.md` |
| «нужен sales / продажник» | `templates/claude-md/sales.md` (опциональный 4-й агент, отдельного урока нет) |
| «как подключить gbrain» | `architecture/02-gbrain-shared-memory.md` + `lessons/lesson-3-agents-with-gbrain/06-setup-gbrain.md` и `07-connect-agents.md` |
| «что такое CLAUDE.md» | `architecture/04-claude-md-guide.md` + `templates/claude-md/README.md` |
| «как сделать telegram-бота для агента» | `architecture/03-telegram-bridge.md` |
| «какие репо нужны помимо этого» | `docs/repo-registry.md` |
| «не нашёл шаблон» | `references/repo-map.md` (карта) |
| «куда положить .mcp.json» | `references/install-paths.md` |
| «claude skill add ругается» | `references/troubleshooting.md` |
| «placeholder `{{agent_name}}` не подставился» | `references/troubleshooting.md` |

## Что я НЕ умею

- Не пишу код за ученика. Я только показываю где взять.
- Не выполняю установку. Команды – в `references/install-paths.md`, копируй и запускай сам.
- Не отвечаю на вопросы вне AgentOS-скиллов. Если про общий стек – переключайся на `senior-brainstorm`, про telegram – на `telegram-bot-builder`.
- Не подменяю чтение README связанных репо. Если ученик деплоит gbrain – ему всё равно надо прочитать README `public-gbrain-agentos`.
