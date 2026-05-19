# Диаграмма: как всё собирается вместе

## Поток установки

```
┌────────────────────────────────────┐
│  YOU (student)                     │
└─────┬──────────────────────────────┘
      │
      │ clones agentos-skills
      ▼
┌────────────────────────────────────┐
│  agentos-skills (this repo)        │
│  • skills (senior-brainstorm…)     │
│  • templates (claude-md/…)         │
│  • lessons (lesson-3-agents…)      │
│  • architecture (this folder)      │
└─────┬──────────────────────────────┘
      │ follow lesson, install:
      ▼
┌────────┬──────────┬──────────┐
│ Layer 1│  Layer 2 │ Layer 3  │
├────────┼──────────┼──────────┤
│ work-  │ shared   │ telegram │
│ space  │ memory   │ bridge   │
│ (PAC)  │ (PGA)    │ (DPC)    │
└─┬──────┴───┬──────┴────┬─────┘
  │          │           │
  │  ▼gbrain MCP         │
  │ ┌──────────────┐     │
  ├─┤ memory       │     │
  ├─┤ recall       │     │
  ├─┤ swarm        │     │
  └─┤ tasks        │     │
    └──────────────┘     │
                         │
     ┌───────────────────┘
     ▼
┌────────────────────────────────────┐
│  Telegram: @your_coder_bot         │
│            @your_marketer_bot      │
│            @your_sales_bot         │
└────────────────────────────────────┘
```

## Что значит каждый блок

- **YOU (student)** – вы, кто проходит урок и собирает AgentOS.
- **agentos-skills** – этот репо. Скиллы (senior-brainstorm, telegram-bot-builder), темплейты CLAUDE.md, уроки, эта архитектурная документация.
- **Layer 1: workspace (PAC)** – локальная директория агента со слойной памятью и хуками. По одной директории на каждого агента.
- **Layer 2: shared memory (PGA)** – self-hosted сервер на отдельном VPS. Postgres + pgvector + 4 MCP endpoint-а.
- **Layer 3: telegram bridge (DPC)** – plugin внутри каждого агента, который слушает Telegram и пробрасывает сообщения в сессию агента.
- **gbrain MCP (memory / recall / swarm / tasks)** – 4 endpoint-а второго слоя. Все агенты подключаются к ним через `.mcp.json`.
- **Telegram bots** – по одному боту на каждого агента. Создаются через @BotFather перед запуском plugin-а.

## Абревиатуры

| Код | Полное название | URL |
|---|---|---|
| **PAC** | public-architecture-claude-code | https://github.com/qwwiwi/public-architecture-claude-code |
| **PGA** | public-gbrain-agentos | https://github.com/qwwiwi/public-gbrain-agentos |
| **DPC** | dashi-plugin-claude-code | https://github.com/qwwiwi/dashi-plugin-claude-code |

## Поток данных в рабочем режиме

После установки всё работает так:

1. Вы пишете боту в Telegram – допустим, coder-у.
2. Plugin (Layer 3) ловит сообщение, пробрасывает в сессию агента.
3. Агент думает в своём workspace (Layer 1), читает локальную память.
4. Если нужен общий контекст – зовёт `recall` через MCP (Layer 2).
5. Если решает важное – сохраняет через `memory.create_decision_note`.
6. Если нужно делегировать – зовёт `swarm.notify(to_agent='marketer')`.
7. Marketer видит уведомление в своём inbox, начинает обработку у себя.
8. Ответ возвращается в Telegram через тот же plugin (Layer 3).

Каждый слой можно отключить, и система останется частично рабочей: без Layer 3 агент работает только в терминале, без Layer 2 агенты теряют общую память, без Layer 1 ничего нет.
