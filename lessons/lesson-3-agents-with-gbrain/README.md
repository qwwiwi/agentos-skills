# Lesson 3: Три агента с общим gbrain

> **Сначала прочитай** [`AGENTIC-SWARM.md`](../../AGENTIC-SWARM.md) в корне репо – это canonical teaching brief: зачем нужен рой, как агенты делят память и работу. Этот урок – практическая реализация описанной там модели.

Боевой урок. За ~90 минут поднимаешь команду из трёх AI-агентов (Homer – coder, Edith – Second Brain, Marketer – контент / growth) и подключаешь их к общей памяти (gbrain – Second Brain поверх Postgres+pgvector). После урока агенты могут отправлять друг другу задачи и читать общую базу решений.

## Цель

Получить рабочий мини-офис из трёх Claude Code агентов, которые:

- работают каждый в своём workspace со своим CLAUDE.md (Homer / Edith / Marketer)
- ходят в общий gbrain через MCP (4 сервера: memory, recall, swarm, tasks)
- умеют `swarm.notify` друг другу (inter-agent сообщения)
- видят общую базу решений через `recall` (полнотекст + семантика)

## Outcomes

После урока у тебя будет:

- три отдельных workspace `~/.claude-lab/<имя>/.claude/` с заполненными CLAUDE.md
- gbrain-сервер на собственном VPS с TLS и Bearer-токенами
- три персональных Bearer-токена в `.mcp.json` каждого агента
- успешный smoke: один агент шлёт сообщение, второй его видит и отвечает
- понимание архитектуры: где живёт память, как агенты её делят, что писать в общую базу, а что в личную

## Кому подходит

- Понимаешь, что такое CLI и git, можешь поставить пакет через `apt`/`brew`/`pip`
- Уже работал с Claude Code хотя бы один раз (видел `~/.claude/`)
- Готов потратить ~12€/мес на VPS под gbrain (либо есть уже свой сервер)

## Кому НЕ подходит

- Никогда не открывал терминал – сначала пройди базовый туториал по CLI
- Не готов разбираться с DNS/TLS/Caddy – здесь это будет (минимально, но будет)
- Хочешь готовое SaaS-решение – gbrain self-hosted, инфра твоя

## Что понадобится (preview)

- Mac или Linux ноутбук (Windows – через WSL2)
- Claude Code установлен и авторизован
- VPS под gbrain (рекомендуется Hetzner CPX42 ~12€/мес, 4 vCPU / 8 GiB)
- Домен с возможностью прописать A-запись
- ~20 GB свободного места на VPS
- Три Telegram-бота (по одному на агента) – опционально, для связи с человеком

Полный чек-лист – в [02-prerequisites.md](./02-prerequisites.md).

## Оценка времени

| Шаг | Минут |
|---|---:|
| 01-02. Подготовка | 10 |
| 03. Homer (coder) | 10 |
| 04. Edith (Second Brain) | 10 |
| 05. Marketer (контент) | 10 |
| 06. gbrain self-host | 30 |
| 07. Подключение MCP | 10 |
| 08. Тест координации | 10 |
| **Итого** | **~90** |

## Структура урока

1. [01-overview.md](./01-overview.md) – зачем три агента, как они взаимодействуют, архитектура на одном слайде
2. [02-prerequisites.md](./02-prerequisites.md) – что установить, какие аккаунты, какие ресурсы
3. [03-create-coder.md](./03-create-coder.md) – создаём Homer (workspace + CLAUDE.md)
4. [04-create-edith.md](./04-create-edith.md) – создаём Edith (Second Brain – raw / wiki / output)
5. [05-create-marketer.md](./05-create-marketer.md) – создаём Marketer (контент, lead-gen, public presence)
6. [06-setup-gbrain.md](./06-setup-gbrain.md) – self-host gbrain на VPS, токены
7. [07-connect-agents.md](./07-connect-agents.md) – подключаем все три workspace к gbrain
8. [08-test-coordination.md](./08-test-coordination.md) – smoke-тест swarm.notify + recall
9. [09-troubleshooting.md](./09-troubleshooting.md) – частые проблемы и решения

## Что делать после урока

- **Подключи свой Telegram-бот к каждому агенту** – используй `dashi-plugin-claude-code` (Telegram bridge). Так агенты смогут говорить с тобой в чате, а не только в терминале
- **Настрой кастомные триггеры** – пропиши в CLAUDE.md специфичные для тебя слова-команды («сделай лендинг», «проверь воронку»)
- **Добавь четвёртого агента** – повтори шаги 03-05, например для sales или для мониторинга (опциональный 4-й агент по swarm-doc)
- **Включи Telegram-инбокс gbrain** – собирай чаты, мемы, заметки в общую память (см. `public-gbrain-agentos` Path B)
- **Поделись своими CLAUDE.md** – если получилось что-то интересное, открой PR в `agentos-skills/templates/claude-md/`
