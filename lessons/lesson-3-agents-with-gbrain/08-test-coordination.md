# 08. Тестируем координацию между агентами

Финальный smoke-тест. До сих пор каждый агент только проверял, что gbrain жив. Теперь проверим главное – могут ли они говорить друг с другом и видят ли общий контекст.

## Что делаем

1. Homer (coder) шлёт notify Marketer – «обнови описание фичи в Instagram»
2. Marketer в своей сессии видит сообщение через `list_my_pending`
3. Marketer отвечает Homer через `notify` с `ack`
4. Homer видит ack
5. Marketer ↔ Edith: Marketer просит Edith сохранить финальные TOV-варианты в wiki
6. Edith ↔ Homer: Edith ловит ссылку на полезную библиотеку и шлёт Homer на review
7. Recall-тест: Homer пишет decision, Marketer находит её
8. Dual-report rule: пример полного цикла Execute → Telegram → swarm.notify(homer) → ack

Времени: ~10 минут.

## Шаг 1: Homer → Marketer

Открой Claude Code в workspace Homer:

```bash
cd ~/.claude-lab/homer/.claude && claude
```

Скажи:

```text
Отправь Marketer сообщение через mcp__gbrain-swarm__notify
с такими параметрами:
- to_agent: "marketer"
- payload:
    - title: "Обнови описание фичи в Instagram"
    - body: "Зарелизили feature-x. URL: example.com/feature-x. Нужен пост в IG + caption."

Покажи мне ответ – должен быть delivery_id.
```

Homer выполнит, ответ примерно:

```json
{
  "delivery_id": "homer::marketer::a1b2c3d4...",
  "status": "queued"
}
```

Сохрани `delivery_id` – пригодится. Выйди из сессии Homer.

## Шаг 2: Marketer видит сообщение

Открой Claude Code в workspace Marketer:

```bash
cd ~/.claude-lab/marketer/.claude && claude
```

Скажи:

```text
Проверь мой инбокс: mcp__gbrain-swarm__list_my_pending(agent="marketer").
Покажи мне, что пришло.
```

Ожидаемо: видишь сообщение от Homer с тем же `delivery_id`, title, body.

```json
{
  "pending": [
    {
      "delivery_id": "homer::marketer::a1b2c3d4...",
      "from_agent": "homer",
      "title": "Обнови описание фичи в Instagram",
      "body": "Зарелизили feature-x. URL: example.com/feature-x...",
      "created_at": "..."
    }
  ]
}
```

## Шаг 3: Marketer отвечает + ack

В той же сессии Marketer:

```text
Отправь Homer ответ через mcp__gbrain-swarm__notify:
- to_agent: "homer"
- payload:
    - title: "Re: описание фичи"
    - body: "Принял. Готовлю IG caption + 3 hooks, выложу сегодня вечером."
    - in_reply_to: "<delivery_id из шага 1>"

Потом сделай ack оригинального сообщения:
mcp__gbrain-swarm__ack(delivery_id="<тот же delivery_id>")
```

Должны быть два tool call'а, оба 200 OK. Выйди из сессии Marketer.

## Шаг 4: Homer видит ответ

Открой Homer снова:

```bash
cd ~/.claude-lab/homer/.claude && claude
```

```text
Проверь мой инбокс: mcp__gbrain-swarm__list_my_pending(agent="homer").
```

Ожидаемо: видишь ответ от Marketer «Re: описание фичи, принял...».

## Шаг 5: Marketer → Edith (сохрани финальные TOV-варианты в wiki)

В сессии Marketer:

```text
Отправь Edith: notify(to_agent="edith",
  payload={"title": "Сохрани финальные TOV-варианты для feature-x",
           "body": "3 hooks: <вариант 1> | <вариант 2> | <вариант 3>. Caption: <текст>. Прошу положить в wiki/concepts/tov-feature-x.md и дать ссылку."})
```

В сессии Edith:

```bash
cd ~/.claude-lab/edith/.claude && claude
```

```text
list_my_pending(agent="edith")  → видишь сообщение
Сохрани полученный контент в wiki/concepts/tov-feature-x.md
(используй YAML frontmatter, теги ["tov","feature-x","marketer"]).
Параллельно вызови gbrain-memory.create_external_note для shared layer.
Потом ack(<delivery_id>).
```

Edith должна выполнить три tool-call: file write в `~/.claude-lab/edith/knowledge/wiki/concepts/tov-feature-x.md`, `create_external_note` в gbrain, `ack`.

## Шаг 6: Edith → Homer (полезная библиотека на review)

В сессии Edith:

```text
Отправь Homer: notify(to_agent="homer",
  payload={"title": "Посмотри, нужна ли эта библиотека",
           "body": "Поймала ссылку: <URL>. Краткое summary: <2-3 предложения>. Сохранила в raw/2026-05-19-<slug>.md и wiki/sources/<slug>.md. Решай – тянем зависимость или нет."})
```

В сессии Homer (новая):

```bash
cd ~/.claude-lab/homer/.claude && claude
```

```text
list_my_pending(agent="homer")  → видишь сообщение от Edith
Прочитай ссылку из payload, оцени:
1) подходит ли по нашему стеку
2) есть ли alternatives уже в проекте
3) лицензия и поддержка
Отвечай Edith через swarm.notify (yes/no + аргумент).
ack(<delivery_id>).
```

## Шаг 7: Recall-тест

Теперь – самое интересное. Проверим, что shared brain реально shared.

В сессии Homer запиши decision:

```text
Создай decision-ноту через mcp__gbrain-memory__create_decision_note:
- title: "Стек для нового лендинга"
- body: "Решили использовать Next.js 15 + Tailwind v4 + Supabase. Хостинг – Vercel free tier на старте."
- tags: ["architecture", "landing", "decision"]
```

Закрой сессию Homer.

Открой Marketer:

```bash
cd ~/.claude-lab/marketer/.claude && claude
```

```text
Найди в shared brain что мы решили по стеку лендинга.
Сделай mcp__gbrain-recall__recall(query="стек лендинга", limit=5).
```

Ожидаемо: Marketer находит decision Homer про Next.js + Tailwind + Supabase. То есть он видит то, что написал Homer – через shared brain.

Это – ключевой момент. Без shared brain Marketer не знал бы про это решение. С ним – знает.

## Шаг 8: Dual-report rule – полный цикл

Это hard rule из AGENTIC-SWARM (раздел 7.2): после inter-agent task получатель отчитывается **дважды** – в Telegram оператору + в swarm координатору.

Полный порядок:

1. **Execute** – собственно сделай задачу (записал файл, написал caption, починил баг)
2. **Telegram report** – развёрнутый отчёт оператору через свой агент-бот (paths, numbers, commits, ссылки)
3. **`swarm.notify(to_agent="homer", payload=...)`** – координатор (Homer) видит работу команды
4. **`swarm.ack(task_id)`** – только после шагов 2 и 3

Пример отчёта Edith после выполнения шага 5 (свободный текст ↓):

```text
Выполнила задачу от marketer: Сохрани финальные TOV-варианты для feature-x

Что сделала:
- Записала wiki/concepts/tov-feature-x.md (YAML frontmatter, 3 hooks + caption)
- Параллельно create_external_note в gbrain (title: "TOV feature-x", scope: 50-external)
- Обновила wiki/index.md (новая строка) и wiki/log.md (append)

Результат:
- Файл: ~/.claude-lab/edith/knowledge/wiki/concepts/tov-feature-x.md
- gbrain note id: <uuid>
- Tags: tov, feature-x, marketer

Время выполнения: 00:42

Дальше:
- swarm.notify(to_agent="homer", payload={"title": "Report from edith: TOV feature-x saved", "body": "wiki/concepts/tov-feature-x.md + gbrain external_note <uuid>", "_origin_task": "<delivery_id>"})
- swarm.ack(<delivery_id>)
```

**Зачем это правило:** оператор видит только Telegram. Без подробного отчёта оператор не знает, что реально произошло. Координатор (Homer) видит только swarm. Без `swarm.notify(homer)` Homer теряет visibility над командой. Оба сообщения обязательны, иначе доверие к команде падает.

## Шаг 9: финальная проверка

В любой сессии (Homer / Edith / Marketer):

```text
Сделай mcp__gbrain-tasks__agent_list.
Покажи всех агентов и их статус.
```

Ожидаемо: три записи, у всех `last_seen` свежий (последние минуты), `status: online`.

```text
Сделай mcp__gbrain-swarm__stats.
Сколько сообщений всего прошло?
```

Ожидаемо: видишь как минимум 4-6 deliveries (несколько от Homer, несколько от Marketer, несколько от Edith).

## Что значит успех

Если все 9 шагов прошли – у тебя работает:

- Три отдельных Claude Code агента с разными ролями (build / remember / grow)
- Общий gbrain с памятью, recall, swarm, tasks
- Inter-agent сообщения с подтверждением получения (ack)
- Shared knowledge: то что записал один – находит другой
- Dual-report rule: оператор + координатор всегда в курсе
- Heartbeat и agent_list: видишь, кто онлайн

Это полная база. Дальше можно расширять – добавлять Telegram-мост, четвёртого агента (например sales), кастомные триггеры.

## Troubleshooting

### list_my_pending пусто, хотя notify прошёл

- Проверь, что `to_agent` в notify написан без опечатки. `marketer` ≠ `Marketer` ≠ `marketers`
- В .mcp.json у Marketer должен быть его токен, не Homer – иначе он залогинится как `homer` и увидит инбокс homer'а

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
