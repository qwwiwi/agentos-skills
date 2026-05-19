# 08. Тестируем координацию между агентами

Финальный smoke-тест. До сих пор каждый агент только проверял, что gbrain жив. Теперь проверим главное – могут ли они говорить друг с другом и видят ли общий контекст.

## Что делаем

1. Хомер (coder) шлёт notify Маркету (marketer) – «лендинг готов»
2. Маркет в своей сессии видит сообщение через `list_my_pending`
3. Маркет отвечает Хомеру через `notify` с `ack: true`
4. Хомер видит ack
5. То же между Coder ↔ Sales
6. То же между Marketer ↔ Sales
7. Recall-тест: один пишет decision, второй находит её

Времени: ~10 минут.

## Шаг 1: Coder → Marketer

Открой Claude Code в workspace Хомера:

```bash
cd ~/.claude-lab/homer/.claude && claude
```

Скажи:

```text
Отправь Маркету сообщение через mcp__gbrain-swarm__notify
с такими параметрами:
- to_agent: "kael"
- payload:
    - title: "Лендинг готов к публикации"
    - body: "URL: example.com/feature-x. Нужен пост в Telegram-канал."

Покажи мне ответ – должен быть delivery_id.
```

Хомер выполнит, ответ примерно:

```json
{
  "delivery_id": "homer::kael::a1b2c3d4...",
  "status": "queued"
}
```

Сохрани `delivery_id` – пригодится. Выйди из сессии Хомера.

## Шаг 2: Marketer видит сообщение

Открой Claude Code в workspace Маркета:

```bash
cd ~/.claude-lab/kael/.claude && claude
```

Скажи:

```text
Проверь мой инбокс: mcp__gbrain-swarm__list_my_pending(agent="kael").
Покажи мне, что пришло.
```

Ожидаемо: видишь сообщение от Хомера с тем же `delivery_id`, title, body.

```json
{
  "pending": [
    {
      "delivery_id": "homer::kael::a1b2c3d4...",
      "from_agent": "homer",
      "title": "Лендинг готов к публикации",
      "body": "URL: example.com/feature-x. Нужен пост в Telegram-канал.",
      "created_at": "..."
    }
  ]
}
```

## Шаг 3: Marketer отвечает ack

В той же сессии Маркета:

```text
Отправь Хомеру ответ через mcp__gbrain-swarm__notify:
- to_agent: "homer"
- payload:
    - title: "Re: Лендинг готов"
    - body: "Принял. Готовлю пост, выложу сегодня вечером."
    - in_reply_to: "<delivery_id из шага 1>"

Потом сделай ack оригинального сообщения:
mcp__gbrain-swarm__ack(delivery_id="<тот же delivery_id>")
```

Должны быть два tool call'а, оба 200 OK. Выйди из сессии Маркета.

## Шаг 4: Coder видит ответ

Открой Хомера снова:

```bash
cd ~/.claude-lab/homer/.claude && claude
```

```text
Проверь мой инбокс: mcp__gbrain-swarm__list_my_pending(agent="homer").
```

Ожидаемо: видишь ответ от Маркета «Re: Лендинг готов, принял...».

## Шаг 5: Coder ↔ Sales

Аналогично. В сессии Хомера:

```text
Отправь Клозеру: notify(to_agent="closer",
  payload={"title": "Новый продукт на лендинге",
           "body": "Цена 49€, описание в example.com/feature-x"})
```

В сессии Клозера:

```text
list_my_pending(agent="closer")  → видишь сообщение
notify(to_agent="homer",
  payload={"title": "Re: Новый продукт",
           "body": "Принял в воронку. Запушу follow-up через 3 дня."})
ack(<delivery_id>)
```

## Шаг 6: Marketer ↔ Sales

В сессии Маркета:

```text
Отправь Клозеру: notify(to_agent="closer",
  payload={"title": "Завтра рассылка по базе",
           "body": "Ожидаем 50-100 новых лидов. Подготовь скрипты."})
```

В сессии Клозера:

```text
list_my_pending(agent="closer")  → видишь два сообщения теперь
ack оба
```

## Шаг 7: Recall-тест

Теперь – самое интересное. Проверим, что shared brain реально shared.

В сессии Хомера запиши decision:

```text
Создай decision-ноту через mcp__gbrain-memory__create_decision_note:
- title: "Стек для нового лендинга"
- body: "Решили использовать Next.js 15 + Tailwind v4 + Supabase. Хостинг – Vercel free tier на старте."
- tags: ["architecture", "landing", "decision"]
```

Закрой сессию Хомера.

Открой Маркета:

```bash
cd ~/.claude-lab/kael/.claude && claude
```

```text
Найди в shared brain что мы решили по стеку лендинга.
Сделай mcp__gbrain-recall__recall(query="стек лендинга", limit=5).
```

Ожидаемо: Маркет находит decision Хомера про Next.js + Tailwind + Supabase. То есть он видит то, что написал Хомер – через shared brain.

Это – ключевой момент. Без shared brain Маркет не знал бы про это решение. С ним – знает.

## Шаг 8: финальная проверка

В любой сессии (Хомер, Маркет или Клозер):

```text
Сделай mcp__gbrain-tasks__agent_list.
Покажи всех агентов и их статус.
```

Ожидаемо: три записи, у всех `last_seen` свежий (последние минуты), `status: online`.

```text
Сделай mcp__gbrain-swarm__stats.
Сколько сообщений всего прошло?
```

Ожидаемо: видишь как минимум 4-6 deliveries (несколько от Хомера, несколько от Маркета, несколько от Клозера).

## Что значит успех

Если все 7 шагов прошли – у тебя работает:

- Три отдельных Claude Code агента с разными ролями
- Общий gbrain с памятью, recall, swarm, tasks
- Inter-agent сообщения с подтверждением получения (ack)
- Shared knowledge: то что записал один – находит другой
- Heartbeat и agent_list: видишь, кто онлайн

Это полная база. Дальше можно расширять – добавлять Telegram-мост, новых агентов, кастомные триггеры.

## Troubleshooting

### list_my_pending пусто, хотя notify прошёл

- Проверь, что `to_agent` в notify написан без опечатки. `kael` ≠ `Kael` ≠ `kaeli`
- В .mcp.json у Маркета должен быть его токен, не Хомера – иначе он залогинится как `homer` и увидит инбокс homer'а

### recall возвращает 0 matches на свежий decision

- Decision только что создан – подожди 5-10 секунд (FastEmbed эмбеддинг занимает время)
- Проверь query – попробуй ближе к тексту decision, например `recall(query="Next.js")` напрямую
- Если и через минуту 0 matches – проверь, что decision действительно записался: `mcp__gbrain-memory__list_decisions`

### swarm.stats показывает только твою сессию

- В Path A установке `stats` агрегирует по всем агентам, но если agent_tokens разные – возможно agent_id определяется по токену. Проверь, что у всех трёх токенов разные identities в `agent_tokens` table

### Один из агентов не отвечает

- Свежая сессия Claude Code в его workspace
- Перепроверь .mcp.json и токен
- Иногда помогает короткий перезапуск: выйти из claude, `claude` снова

## Готово

Три агента работают. Если что-то не заработало – переходи к [09-troubleshooting.md](./09-troubleshooting.md). Если всё хорошо – вернись в README урока, секция «Что дальше».
