# Telegram Mini Apps

Mini App – веб-приложение, открытое внутри Telegram. Для нас это почти всегда та же сборка
кабинета, что и на сайте, но со своим входом и своими правилами экрана. Правила ниже – из
живых мини-аппов (кабинеты практикума, AgentOS, EdgeLab Space). Источники – в
[owner-corrections.md](owner-corrections.md).

Документация: https://core.telegram.org/bots/webapps (сверено 04.11.2026, Bot API 10.3).

## 1. Архитектура

- **Отдельная сборка или поддомен с флагом «только Telegram».** Веб-кабинет (`platform.example.com`) и мини-апп (`app.example.com`) – одна кодовая база, флаг на этапе сборки. Сессии на разных поддоменах изолированы (localStorage привязан к origin), поэтому мини-апп не ломает вход на сайте.
- **SDK.** Либо `<script src="https://telegram.org/js/telegram-web-app.js">` (объект `window.Telegram.WebApp`), либо `@tma.js/sdk-react` – он читает параметры запуска сам, без внешнего скрипта. Скрипт грузить только в сборке мини-аппа.
- **Next.js:** если флаг `NEXT_PUBLIC_*` не задан при сборке, мёртвая ветка с SDK остаётся в бандле. Дай значение по умолчанию в `next.config.ts`: `env: { NEXT_PUBLIC_TELEGRAM_ONLY: process.env.NEXT_PUBLIC_TELEGRAM_ONLY ?? "false" }`.
- **Telegram Web держит мини-апп в iframe.** `X-Frame-Options: DENY` даёт пустой экран. Для сборки мини-аппа: `Content-Security-Policy: frame-ancestors 'self' https://web.telegram.org https://*.telegram.org`. Веб-сборке запрет оставить.
- **Вход в мини-апп** – кнопка меню бота (BotFather → Menu Button) или прямая ссылка `t.me/<bot>/<app>`. Без initData (открыли в браузере) – экран «Откройте в Telegram».

## 2. Вход: initData проверяет только сервер

Клиент отправляет сырую строку `initData` на бэкенд, бэкенд проверяет подпись и выдаёт свою сессию.
`initDataUnsafe` на клиенте – только для отображения, никогда для решений о доступе.

Алгоритм (официальный):
1. Разобрать query-строку, вынуть `hash`.
2. data-check-string = все остальные пары `key=value`, отсортированные по ключу, через `\n`.
3. `secret_key = HMAC_SHA256(key="WebAppData", msg=bot_token)`.
4. Сравнить `hex(HMAC_SHA256(key=secret_key, msg=data_check_string))` с `hash` за постоянное время.
5. Проверить свежесть `auth_date` (у нас – 15 минут).

```python
import hashlib
import hmac
import json
import time
from urllib.parse import parse_qsl

MAX_AGE_SECONDS = 900


def verify_init_data(init_data: str, bot_token: str) -> dict | None:
    """Verify Telegram Mini App initData.

    Args:
        init_data: Raw `Telegram.WebApp.initData` query string from the client.
        bot_token: Token of the bot that opened the Mini App.

    Returns:
        The parsed `user` object if the signature is valid and fresh, else None.
    """
    pairs = dict(parse_qsl(init_data, keep_blank_values=True))
    received = pairs.pop("hash", None)
    if not received:
        return None
    check_string = "\n".join(f"{k}={v}" for k, v in sorted(pairs.items()))
    secret = hmac.new(b"WebAppData", bot_token.encode(), hashlib.sha256).digest()
    expected = hmac.new(secret, check_string.encode(), hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected, received):
        return None
    if time.time() - int(pairs.get("auth_date", "0")) > MAX_AGE_SECONDS:
        return None
    return json.loads(pairs["user"]) if "user" in pairs else None
```

```typescript
import { createHmac, timingSafeEqual } from "node:crypto";
import { z } from "zod";

const MAX_AGE_SECONDS = 900;
const TelegramUser = z.object({
  id: z.number().int(),
  first_name: z.string(),
  last_name: z.string().optional(),
  username: z.string().optional(),
});
export type TelegramUser = z.infer<typeof TelegramUser>;

export function verifyInitData(initData: string, botToken: string): TelegramUser | null {
  const params = new URLSearchParams(initData);
  const hash = params.get("hash");
  if (!hash) return null;
  params.delete("hash");
  const checkString = [...params.entries()]
    .sort(([a], [b]) => (a < b ? -1 : a > b ? 1 : 0))
    .map(([k, v]) => `${k}=${v}`)
    .join("\n");
  const secret = createHmac("sha256", "WebAppData").update(botToken).digest();
  const expected = createHmac("sha256", secret).update(checkString).digest("hex");
  const a = Buffer.from(expected);
  const b = Buffer.from(hash);
  // timingSafeEqual throws on different lengths: check first.
  if (a.length !== b.length || !timingSafeEqual(a, b)) return null;
  const authDate = Number(params.get("auth_date") ?? 0);
  if (Date.now() / 1000 - authDate > MAX_AGE_SECONDS) return null;
  const parsed = TelegramUser.safeParse(JSON.parse(params.get("user") ?? "null"));
  return parsed.success ? parsed.data : null;
}
```

Ловушки входа (все случались у нас):
- **Сервис проверки подписи должен знать токен того же бота, который открывает мини-апп.** Заглушка вместо токена в `.env` API даёт «Invalid signature» у всех.
- **Лимит запросов на вход – по Telegram ID после проверки подписи**, а не только по IP: за NAT и Cloudflare у многих людей один IP. За Cloudflare IP брать из `CF-Connecting-IP`.
- **На клиенте – один запрос входа за раз**, без повтора, пока первый не ответил; на 429 – экран «Подождите» и один повтор с паузой. Иначе каждое открытие мини-аппа устраивает «шторм» запросов.
- Параметр запуска `tgWebAppData` в адресной строке – это учётные данные: забрать в память и убрать из URL (`history.replaceState`) до всего остального.

## 3. Экран: правила для каждого маршрута

| Что | Как | Почему |
|---|---|---|
| Готовность | `WebApp.ready()` и `WebApp.expand()` при старте | убрать заглушку загрузки, открыть на всю высоту |
| **Свайп вниз** | `WebApp.disableVerticalSwipes()` **на каждом маршруте**, не только на одном экране (Bot API 7.7+; в `@tma.js` – `swipeBehavior.disableVertical`) | владелец: мини-апп закрывается только кнопкой, никогда случайным свайпом |
| Назад | `BackButton.show()` на всех экранах кроме главного, `onClick` → история роутера; открытое модальное окно закрывается кнопкой «Назад», а не уходит с экрана | системная кнопка Telegram и Android-«назад» должны вести себя как в приложении |
| Тема | цвета из `themeParams` (`bg_color`, `text_color`), подписка на событие `themeChanged`; `setHeaderColor` в цвет фона | мини-апп выглядит родным в светлой и тёмной теме Telegram |
| Высота | `viewportStableHeight` и `visualViewport` → CSS-переменная `--app-height`; события `viewportChanged` | клавиатура и шторка Telegram меняют высоту |
| Safe area | см. ниже | |

### Safe area – учитывать один раз

У Telegram два набора отступов: `safeAreaInset` (вырез, полоска «домой») и `contentSafeAreaInset`
(элементы интерфейса Telegram), плюс браузерный `env(safe-area-inset-*)`. Если сложить всё,
отступ получается двойным и меню «висит» над краем.

Как сделано у нас: один набор CSS-переменных, в каждой – **максимум**, а не сумма.

```typescript
// Called on start and on safeAreaChanged / contentSafeAreaChanged / viewportChanged.
function setInsets(safe?: Insets, content?: Insets): void {
  const root = document.documentElement;
  for (const edge of ["top", "bottom", "left", "right"] as const) {
    root.style.setProperty(`--tg-safe-${edge}`, `${Math.max(safe?.[edge] ?? 0, content?.[edge] ?? 0)}px`);
  }
}
```

```css
:root {
  --safe-bottom: max(env(safe-area-inset-bottom, 0px), var(--tg-safe-bottom, 0px));
  --safe-top: max(env(safe-area-inset-top, 0px), var(--tg-safe-top, 0px));
}
```

Дальше вся вёрстка использует только `--safe-*`. Ни один компонент не добавляет `env(safe-area-inset-*)` сам.
Telegram отдаёт и свои CSS-переменные (`--tg-safe-area-inset-bottom`, `--tg-content-safe-area-inset-bottom`) –
выбери один источник и не смешивай.

## 4. Нижнее меню – плавающая капсула

Решение владельца 04.11.2026: нижнее меню кабинета и мини-аппа – как в YouTube и Instagram.

- Плавающая капсула: `position: fixed`, отступ 16 px от краёв и от `--safe-bottom`, `border-radius: 999px`, ширина колонки контента (до ~560 px) на широком экране.
- Высота около **54 px**, внутри сетка из равных вкладок (`grid-template-columns: repeat(N, 1fr)`), внутренний отступ 3 px.
- **Только иконки, без подписей.** Подпись – в `aria-label`, активная вкладка – `aria-current="page"`.
- Активная вкладка – плашка-пилюля **почти на всю высоту** капсулы (около 46 px), полное скругление.
- **Непрозрачный фон** (цвет фона страницы, прозрачность около 0.98) + тонкая рамка и мягкая тень. Полупрозрачное меню поверх текста читается плохо.
- Иконки 26 px, на нажатие – лёгкое уменьшение (`scale(.94)`); во вкладке профиля – аватар.
- **Прячется при прокрутке вниз**, появляется при прокрутке вверх (`transform: translateY(...)`, 200 ms). При фокусе с клавиатуры не прячется. При `prefers-reduced-motion` – без анимации.
- Контент страницы получает нижний отступ на высоту меню + `--safe-bottom`, а `scroll-padding-bottom` не даёт якорям и фокусу уезжать под меню.
- Это единственное исключение из правила скруглений проекта: всё остальное – обычные токены.

```css
.tabbar {
  position: fixed;
  left: calc(16px + var(--safe-left));
  right: calc(16px + var(--safe-right));
  bottom: calc(16px + var(--safe-bottom));
  max-width: 560px;
  margin: 0 auto;
  z-index: 40;
  background: rgba(247, 245, 240, 0.98);
  border: 1px solid rgba(16, 16, 16, 0.12);
  border-radius: 999px;
  box-shadow: 0 4px 24px rgba(16, 16, 16, 0.12);
  transition: transform 200ms ease;
}
.tabbar[data-hidden="true"] { transform: translateY(calc(100% + 32px + var(--safe-bottom))); }
.tabbar:has(:focus-visible) { transform: none; }
.tabbar-inner { display: grid; grid-template-columns: repeat(4, 1fr); height: 52px; padding: 3px; }
.tab { display: flex; align-items: center; justify-content: center; position: relative; }
.tab-highlight { position: absolute; inset: 0; border-radius: 999px; background: var(--accent-soft); }
@media (prefers-reduced-motion: reduce) { .tabbar, .tabbar[data-hidden="true"] { transition: none; transform: none; } }
```

## 5. Как вернуть данные и человека в бота

Оба «очевидных» способа врут:
- **`WebApp.sendData`** определён всегда, даже там, где не работает, и не бросает ошибку. Работает только у мини-аппа, открытого кнопкой обычной клавиатуры (`KeyboardButton(web_app=...)`) в личке. Проверка `if (!tg.sendData)` бесполезна, «успех» после вызова – ложный.
- **`openTelegramLink`** на Android молча глушится, если с последнего касания прошло больше ~10 секунд; на iOS ссылка `?start=` закрывает мини-апп. А `?start=` к уже запущенному боту ничего не доставляет, пока человек не нажмёт START.

Что работает: **очередь**. Мини-апп пишет в API, API кладёт строку в таблицу, бот сам разгребает её и
отправляет сообщение. Две детали: миграция помечает старые строки доставленными (иначе первый проход
разошлёт архив), воркер не шлёт строки старше пары часов.

## 6. Проверка

- Playwright покрывает всё, что доступно без Telegram: маршруты, экран «Откройте в Telegram», ответы 401/404, вёрстку на 390 px.
- Вход по initData, свайпы, BackButton и safe-area проверяются только на телефоне внутри Telegram – финальная проверка за владельцем продукта, на iOS и Android.
- После деплоя убедись, что обе сборки (сайт и мини-апп) на одной версии: частая ошибка – выкатить только сайт.

## 7. Чек-лист Mini App

- [ ] Подпись `initData` проверяет сервер, `auth_date` не старше 15 минут.
- [ ] Лимит входа – по Telegram ID после проверки, на клиенте один запрос входа за раз.
- [ ] `ready()` и `expand()` при старте.
- [ ] `disableVerticalSwipes()` на каждом маршруте.
- [ ] BackButton на всех экранах кроме главного, модальные окна закрываются им.
- [ ] Тема из `themeParams` + `themeChanged`, `setHeaderColor`.
- [ ] Safe area – одна переменная, максимум, не сумма.
- [ ] Нижнее меню – капсула 54 px, иконки с `aria-label`, непрозрачный фон, прячется при прокрутке.
- [ ] `frame-ancestors` для Telegram Web только в сборке мини-аппа.
- [ ] Данные в бота – через очередь, не через `sendData`/`openTelegramLink`.
- [ ] Сайт и мини-апп выкачены одной версией.
