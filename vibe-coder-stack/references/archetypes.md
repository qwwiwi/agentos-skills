# Архетипы: стек, стартовые репозитории, промпты по шагам

Репозитории проверены 4 ноября 2026 через GitHub API: живые, не в архиве. Звёзды – ориентир, не критерий:
у хорошего проекта смотри на коммиты, контрибьюторов и релизы.

Промпты рассчитаны на Claude Code или Codex. Квадратные скобки – подставь своё. Один промпт – один шаг;
после каждого шага смотри результат в браузере и делай коммит.

---

## 1. Продающий лендинг

Одностраничник без кабинета: быстро грузится, попадает в поиск, даёт красивое превью ссылки.

- **Рендер:** SSG.
- **Стек:** Astro + Tailwind. Для совсем простого – чистый HTML + CSS без сборки.
- **Хостинг:** статика за веб-сервером (Caddy/nginx) на своём VPS или Cloudflare Pages.
- **Оплата:** готовый виджет платёжки или ссылка на оплату. Своя платёжная логика на лендинге не нужна.

| Репозиторий | Зачем |
|---|---|
| [withastro/astro](https://github.com/withastro/astro) | сам фреймворк |
| [arthelokyo/astrowind](https://github.com/arthelokyo/astrowind) | бесплатный шаблон лендинга на Astro + Tailwind (бывший onwidget/astrowind) |

```bash
npm create astro@latest my-landing
```

**Как у нас:** лендинг практикума первые недели жил как статический HTML без шага сборки
(`site/` раздаётся как есть), с локальной проверкой битых ссылок и чистых URL. Когда страниц стало
больше – перевели на Astro + Tailwind. Статичный лендинг партнёрского проекта – тот же подход:
HTML в репозитории, выкладка `git archive` + `rsync`.

**Промпты по шагам:**

1. «Собери продающий лендинг на Astro + Tailwind: hero, оффер, программа, отзывы, FAQ, кнопка CTA на [ссылку]. Тексты возьми из [файла с текстами] дословно, ничего не придумывай. SSG, адаптив 320–1440 px.»
2. «Сделай DESIGN.md: палитра, максимум три шрифта с ролями, два значения скругления, кнопки, отступы. Приведи лендинг строго к нему.»
3. «Добавь OG-превью (og:title, og:description, og:image 1200×630), favicon, robots.txt, sitemap. Проверь, что нет горизонтальной прокрутки на 320 и 390 px.»
4. «Напиши scripts/check.sh: сборка, проверка битых ссылок, проверка, что все страницы собрались. Подключи его в .githooks/pre-push.»
5. «Напиши scripts/deploy.sh: отказ при грязном дереве и при коммите, которого нет на origin; сборка из коммита; rsync на [сервер]; после выката – скриншоты живой страницы на 390 и 1440 px.»

---

## 2. SEO-портал / блог

Открытый каталог статей и гайдов. Контент должен пробиваться в поиск.

- **Рендер:** SSG.
- **Стек:** Astro, статьи в markdown прямо в репозитории; для документации – Starlight.
- **Хостинг:** как у лендинга.

| Репозиторий | Зачем |
|---|---|
| [withastro/astro](https://github.com/withastro/astro) | фреймворк |
| [satnaing/astro-paper](https://github.com/satnaing/astro-paper) | SEO-friendly тема блога, самый популярный старт |
| [withastro/starlight](https://github.com/withastro/starlight) | официальный движок документации на Astro |

**Промпты по шагам:**

1. «Собери SEO-портал гайдов на Astro: главная, каталог статей, страница статьи из markdown в content/. SSG, sitemap, RSS.»
2. «Добавь каждой статье title, description, canonical, OG-картинку и разметку Article (JSON-LD).»
3. «Сделай поиск по статьям на клиенте без внешних сервисов.»
4. Деплой – как шаги 4–5 лендинга.

---

## 3. Платформа с личным кабинетом

Вход, регистрация, данные пользователя, оплата, доступы.

- **Рендер:** кабинет за логином – CSR или SSR; публичные страницы (тарифы, «о продукте») – SSG/SSR.
- **Фронт:** Next.js (App Router, `output: 'standalone'`) – если есть публичные страницы или серверный рендер. React + Vite – если всё за логином.
- **Бэкенд:** self-host Supabase (Postgres + Auth + авто-API + RLS). Бизнес-логику, которую не покрывают права RLS, – в тонкий API-сервис (TypeScript: Hono; Python: FastAPI).
- **Валидация:** Zod на каждой границе (форма, API, ответ внешнего сервиса).
- **Хостинг:** VPS, Node-процесс под pm2 или systemd, Caddy впереди (сам выпускает HTTPS-сертификаты).

| Репозиторий | Зачем |
|---|---|
| [vercel/next.js](https://github.com/vercel/next.js) | фреймворк и папка `examples/` |
| [nextjs/saas-starter](https://github.com/nextjs/saas-starter) | официальный SaaS-стартер на Next.js: вход, оплата, кабинет |
| [vitejs/vite](https://github.com/vitejs/vite) | сборщик для варианта React + Vite |
| [TanStack/router](https://github.com/TanStack/router) | TanStack Router и TanStack Start |
| [mwolf1989/tanstack-starter](https://github.com/mwolf1989/tanstack-starter) | TanStack Start + Supabase + shadcn (community, маленький) |
| [t3-oss/create-t3-app](https://github.com/t3-oss/create-t3-app) | эталонный типобезопасный fullstack-стартер, для сравнения подхода |
| [supabase/supabase](https://github.com/supabase/supabase) | Supabase, папка `docker/` – self-host |

Self-host Supabase поднимается по официальному гайду: https://supabase.com/docs/guides/self-hosting/docker.
Цена self-host: бэкапы, обновления и аптайм – на тебе.

**Как у нас:** кабинет AgentOS и платформа EdgeLab Space – Next.js App Router в режиме `standalone`
на self-host Supabase; сборка на машине разработчика, на сервер уезжает готовый `server.js` со статикой,
процесс под pm2, Caddy впереди. Кабинет практикума – та же кодовая база, отдельные маршруты.

**Промпты по шагам:**

1. «Собери кабинет на Next.js App Router (TypeScript strict, без any): вход по email-коду, профиль, [мои сущности]. Данные – self-host Supabase, права – RLS. Типы генерируй из схемы, на всех границах Zod.»
2. «Опиши схему в миграциях SQL (каждая миграция – отдельный файл, повторный запуск безопасен). Для каждой таблицы – политика RLS. Напиши интеграционные тесты на права: чужой пользователь не видит чужие данные.»
3. «Сделай DESIGN.md и приведи кабинет к нему. На телефоне главные кнопки закреплены внизу с отступом под safe-area.»
4. «Подключи оплату через [платёжку]: вебхук проверяет подпись, доступ выдаётся только после подтверждения от платёжки, повторный вебхук не выдаёт доступ второй раз.»
5. «Напиши scripts/check.sh (typecheck, lint, тесты, build) и .githooks/pre-push. Напиши deploy.sh: сборка standalone из чистого коммита, выкладка в releases/<sha>, переключение симлинка current, рестарт процесса, откат на прошлый релиз одной командой.»

---

## 4. Telegram-бот

Отдельная программа на сервере. Полный разбор – в скилле `telegram-bot-builder`.

- **TS-путь:** grammY (Node.js или Bun) – один язык со всем веб-стеком.
- **Python-путь:** aiogram 3 – зрелый async-фреймворк, ближе к AI и данным.
- **Данные:** тот же Postgres / Supabase, что и у кабинета.
- **Хостинг:** зарубежный VPS (с российских серверов `api.telegram.org` часто недоступен), процесс под systemd.

| Репозиторий | Зачем |
|---|---|
| [grammyjs/grammY](https://github.com/grammyjs/grammY) | TS-фреймворк |
| [aiogram/aiogram](https://github.com/aiogram/aiogram) | Python-фреймворк |
| [wakaree/aiogram_bot_template](https://github.com/wakaree/aiogram_bot_template) | шаблон aiogram 3 + PostgreSQL + SQLAlchemy + alembic |

**Промпты по шагам:**

1. «Собери Telegram-бота на [grammY / aiogram 3]: /start с приветствием, /help, [моя логика]. Токен только из .env, глобальный обработчик ошибок, логи без токена.»
2. «Тексты бота вынеси в один файл. Правила: тире «–», кавычки «», короткие абзацы с пустой строкой, эмодзи не больше 1–2 на сообщение.»
3. «Данные – в Postgres. Состояния диалогов (FSM) – не в памяти процесса: деплой не должен обрывать сценарий.»
4. «Сделай systemd-юнит и deploy.sh: выкладка из чистого коммита, рестарт, проверка, что процесс запущен из нового релиза.»

---

## 5. Telegram Mini App (кабинет внутри Telegram)

Тот же фронт кабинета, открытый кнопкой бота внутри Telegram. Вход – по `initData`, без пароля.
Полный разбор – в скилле `telegram-bot-builder`, файл `references/mini-apps.md`.

- **Стек:** фронт кабинета (Next.js или React + Vite) + Telegram SDK (`telegram-web-app.js` или `@tma.js/sdk-react`).
- **Бэкенд:** проверка подписи `initData` на сервере, выдача своей сессии.
- **Сборка:** отдельная сборка или поддомен с флагом «только Telegram», веб-версия кабинета не меняется.

| Репозиторий | Зачем |
|---|---|
| [Telegram-Mini-Apps/tma.js](https://github.com/Telegram-Mini-Apps/tma.js) | SDK для Mini Apps (`@tma.js/sdk`, `@tma.js/sdk-react`) |
| [Telegram-Mini-Apps/reactjs-template](https://github.com/Telegram-Mini-Apps/reactjs-template) | шаблон React + Vite (последний коммит – конец 2025, проверь версии) |
| [Telegram-Mini-Apps/nextjs-template](https://github.com/Telegram-Mini-Apps/nextjs-template) | шаблон Next.js (последний коммит – конец 2025, проверь версии) |

**Промпты по шагам:**

1. «Подключи Telegram Mini App к кабинету: сборка с флагом TELEGRAM_ONLY, вход по initData (подпись проверяет сервер, срок auth_date ограничен), без initData – экран "Откройте в Telegram".»
2. «Экран Mini App: ready() и expand() при старте, свайп вниз для закрытия отключён на всех экранах, BackButton на всех экранах кроме главного, цвета из themeParams, отступы safe-area учтены один раз.»
3. «Нижнее меню – плавающая капсула как в YouTube и Instagram: иконки без подписей с aria-label, высота около 54 px, активная плашка почти на всю высоту, непрозрачный фон, отступ от safe-area, прячется при прокрутке вниз.»
4. «Проверь в Playwright все экраны, которые доступны без Telegram (маршруты, экран "Откройте в Telegram", 401). Финальную проверку входа делаю сам с телефона.»
