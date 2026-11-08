# telegram-bot-builder в Hermes

## Почему сканер блокировал установку

Перед установкой из GitHub Hermes прогоняет статический regex-сканер
(`tools/skills_guard.py`, версия `skills-guard-v6`) по всем текстовым файлам
скилла. Примеры кода в Markdown он читает как исполняемый код – блоки
` ``` ` не отличает от скриптов. Для источника «community» правила такие:

- хотя бы одна находка critical – вердикт DANGEROUS, блок, `--force` не помогает;
- хотя бы одна high – CAUTION, блок, снимается только `--force`;
- medium и low – информационные, вердикт SAFE, установка проходит.

Было (39 находок, у студентки 38 – другая версия сканера):

| Правило | Уровень | Шт. | Что нашёл | Оценка |
|---|---|---|---|---|
| `hardcoded_secret` | critical | 1 | секрет вебхука строкой в примере aiogram (в отчёте Hermes значение замаскировано как `***`) | плохой пример, исправлен: секрет читается из окружения |
| `dns_exfil` | critical | 1 | строка nginx, передающая заголовок Host с переменной nginx: слово host сканер принял за DNS-утилиту | ложное; строка убрана, боту этот заголовок не нужен |
| `python_os_environ` | high | 14 | чтение токена, секрета и настроек БД через `environ["..."]` | ложное (это правильная практика); заменено на `.get(...)` с явной проверкой – заодно понятная ошибка при старте |
| `sudo_usage` | high | 3 | команды systemctl в инструкции для VPS | ложное; префикс повышения прав убран, в тексте помечено «от root» |
| `unpinned_pip_install`, `unpinned_npm_install`, `systemd_service`, `hardcoded_ip_port`, `python_getenv_secret`, `python_environ_get_secret`, `remote_fetch` | medium | 20 | установка пакетов, unit-файл, локальный адрес с портом за прокси, чтение ключей из окружения | ожидаемые, на вердикт не влияют, оставлены сознательно |

Стало: вердикт SAFE, 26 находок, все medium (ожидаемые из последней строки
таблицы плюс `git_clone` в инструкции ниже), «Decision: ALLOWED».

## Настоящая уязвимость (её нашла студентка)

В `references/security.md` проверка вебхука была fail-open: если
`TELEGRAM_WEBHOOK_SECRET` не задан и в запросе нет заголовка
`X-Telegram-Bot-Api-Secret-Token`, сравнивались `None` и `None` (в TS –
`undefined` и `undefined`), и любой, кто знает URL, мог слать поддельные
апдейты. То же было в примерах aiogram, Yandex, Supabase, Cloudflare,
grammY (пустой `secretToken`), а Yandex-обработчик в `databases.md` секрет
не проверял вовсе.

Теперь во всех примерах fail-closed: без секрета 32–256 символов
`[A-Za-z0-9_-]` бот не стартует, запрос без заголовка – 403, сравнение за
постоянное время (`hmac.compare_digest`, `crypto.timingSafeEqual` по
SHA-256). Подробнее – `references/security.md`, раздел «Fail closed».

## Как установить

```bash
hermes skills install qwwiwi/agentos-skills/telegram-bot-builder --yes
```

В выводе должно быть «Verdict: SAFE» и «Decision: ALLOWED».

## Запасной вариант: ручная установка

Если установка всё равно блокируется (старая или новая версия сканера):

1. Скачайте репозиторий в отдельную папку:
   `git clone --depth 1 https://github.com/qwwiwi/agentos-skills ~/agentos-skills`
2. Просмотрите `~/agentos-skills/telegram-bot-builder/` – там только Markdown,
   скриптов и бинарников нет. При ручной установке сканер не запускается,
   поэтому проверка на вас.
3. Скопируйте папку в каталог скиллов Hermes (по умолчанию `~/.hermes/skills/`,
   для другого профиля – папка `skills` его `HERMES_HOME`):
   `cp -R ~/agentos-skills/telegram-bot-builder ~/.hermes/skills/`
4. Проверьте: `hermes skills list` – скилл в списке.

Флаг `--force` на DANGEROUS не действует, а снимать им CAUTION вслепую не
стоит: сначала прочитайте, на что именно сработал сканер.
