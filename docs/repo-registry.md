# Связанные публичные репозитории

## TL;DR

`agentos-skills` – не одинокий репозиторий. Это часть набора публичных проектов под одним owner'ом (`qwwiwi`), которые вместе складываются в полноценный стек AgentOS: личные агенты, их общая память, мост в Telegram и шаблоны настройки. Здесь – карта пяти репозиториев, которые ученик скорее всего захочет открыть после этого.

## Карта репозиториев

### 1. qwwiwi/agentos-skills (этот репозиторий)

– публичный, MIT
– Skills, шаблоны CLAUDE.md, уроки и архитектурные конспекты
– URL: https://github.com/qwwiwi/agentos-skills
– Use case: «дай мне готовые навыки и шаги, чтобы поднять агентов с нуля»

### 2. qwwiwi/public-architecture-claude-code

– публичный, ~6 stars
– Universal architecture для Claude Code: слойная локальная память, hooks, self-improvement, семантический поиск
– URL: https://github.com/qwwiwi/public-architecture-claude-code
– Use case: «создай мне один workspace под конкретную роль (coder / marketer / sales)»; интерактивный `install.sh` спрашивает имя, роль, модель

### 3. qwwiwi/public-gbrain-agentos

– публичный, Apache 2.0
– Second Brain: self-host shared memory (MCP + Postgres + pgvector), inbox-agent и генератор workspace'ов
– URL: https://github.com/qwwiwi/public-gbrain-agentos
– Use case: «подними один сервер, который станет общей памятью для всех агентов сразу»; внутри – четыре MCP сервера (memory / recall / swarm / tasks)

### 4. qwwiwi/dashi-plugin-claude-code

– публичный, CANONICAL Telegram bridge
– Jarvis Channel – custom Claude Code channel plugin для Telegram (Bun + TypeScript)
– URL: https://github.com/qwwiwi/dashi-plugin-claude-code
– Use case: «хочу управлять агентами с телефона через Telegram бота»; ставится отдельно на каждого агента

### 5. qwwiwi/edgelab-claude-md

– публичный, ~2 stars
– Полное руководство по заполнению `CLAUDE.md` для AI-агентов на Claude Code, два уровня: глобальный и проектный
– URL: https://github.com/qwwiwi/edgelab-claude-md
– Use case: «объясни мне зачем в `CLAUDE.md` каждый блок и как его заполнять под себя»

### 6. qwwiwi/jarvis-telegram-gateway (DEPRECATED)

– публичный, ~6 stars
– Статус: deprecated, не использовать в новых установках
– Cutover на dashi-plugin-claude-code: 2026-06-15
– Архивируется: 2026-09-15
– URL: https://github.com/qwwiwi/jarvis-telegram-gateway
– Use case: только историческая справка – если читаешь старые статьи и видишь упоминание gateway, знай, что заменено

## Что НЕ упомянуто

Все приватные репозитории владельца сюда не входят – они либо служебные, либо содержат рабочие данные. Если в каком-то источнике упомянут репо, которого ты не нашёл в этой карте – с высокой вероятностью он приватный, и доступа без приглашения не будет. Не пытайся подобрать токен или клонировать через хитрости – собери стек из перечисленных шести репозиториев, этого достаточно.

## Как они связаны

```
                       ┌────────────────────────┐
                       │   agentos-skills (this)│
                       │   skills + templates   │
                       │   + lessons + arch     │
                       └───────────┬────────────┘
                                   │ ссылается на
            ┌──────────────────────┼──────────────────────┐
            ▼                      ▼                      ▼
  ┌─────────────────┐   ┌─────────────────────┐  ┌──────────────────┐
  │ public-arch-... │   │ public-gbrain-...   │  │ edgelab-claude-md│
  │ workspace gen   │   │ shared memory       │  │ CLAUDE.md guide  │
  │ (per agent)     │   │ (one for all)       │  │ (reference)      │
  └────────┬────────┘   └──────────┬──────────┘  └──────────────────┘
           │                       │
           │ создаёт               │ хранит общую память
           ▼                       ▼
  ┌─────────────────────────────────────────┐
  │   твои агенты (coder / marketer / sales)│
  │   каждый – отдельный Claude Code        │
  └────────────────┬────────────────────────┘
                   │ управляются через
                   ▼
        ┌────────────────────────┐
        │ dashi-plugin-claude-... │
        │ Telegram bridge         │
        └────────────────────────┘
```

## Дальше

Полные TL;DR каждого репозитория и use cases по фазам – в нашей папке `architecture/`. Если хочешь увидеть всё в действии – собери стек по уроку `lessons/lesson-3-agents-with-gbrain/`.
