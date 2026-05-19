# Что должно быть установлено

Прежде чем браться за уроки и шаблоны из этого репозитория – пройди этот чек-лист. Без перечисленных инструментов и аккаунтов всё равно упрёшься в стену на первом же шаге.

## Системные требования

– **OS**: macOS (Apple Silicon или Intel), Linux (Ubuntu 22.04+ / Debian 12+), Windows через WSL2 (Ubuntu внутри). На голом Windows без WSL2 ничего не заработает – Claude Code и bun завязаны на POSIX
– **RAM**: 8 GB минимум для запуска агентов локально (Claude Code сам по себе ест 1–2 GB на агента). На VPS под gbrain – 4 GB достаточно
– **Disk**: 5 GB свободно для workspace'ов всех агентов плюс память. На VPS под gbrain – 20 GB (Postgres + pgvector + логи + бэкапы)
– **Сеть**: стабильный канал, в идеале без NAT, иначе Telegram webhook не пробьётся. Если NAT неизбежен – используй reverse SSH tunnel или Tailscale

## Инструменты

### claude (Claude Code CLI)

Сердце всего. Без него ничего не запустится.

– Источник: https://claude.com/claude-code
– Установка: следуй инструкции на странице (npm install или официальный installer)
– Проверка: `claude --version`

### gh (GitHub CLI)

Нужен для клонирования private шаблонов, открытия issues и проверки скиллов.

– macOS: `brew install gh`
– Ubuntu / Debian: `apt install gh` (или официальный repo если apt версия устарела)
– Windows (WSL2): то же что Ubuntu
– После установки: `gh auth login` – пройди OAuth

### git

Скорее всего уже есть.

– macOS: `xcode-select --install` или `brew install git`
– Linux: `apt install git`
– Проверка: `git --version` (минимум 2.30)

### Node 22+

Нужен для сборки `dashi-plugin-claude-code` (Telegram bridge).

– Рекомендую через nvm: `curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash`
– Затем: `nvm install 22 && nvm use 22`
– Проверка: `node --version` → `v22.x.x`

### Python 3.12+

Нужен для дополнительных скриптов в шаблонах и lesson hooks.

– macOS: `brew install python@3.12`
– Ubuntu: `apt install python3.12 python3.12-venv`
– Проверка: `python3 --version` → `3.12.x` или выше

### Bun

Runtime для Telegram плагина. Быстрее Node, и плагин на нём собран.

– Все OS: `curl -fsSL https://bun.sh/install | bash`
– Затем перезапусти shell или `source ~/.zshrc`
– Проверка: `bun --version`

## Аккаунты

### GitHub

– Бесплатный план достаточно
– Нужно SSH key добавить (`ssh-keygen -t ed25519` если нет, потом `gh ssh-key add`)

### Anthropic

– Лучший вариант: подписка Anthropic Max ($200/мес) – без per-token биллинга, можно гонять агентов сутками
– Альтернатива: Anthropic Pro ($20/мес) + API key с балансом ($30–100/мес в зависимости от интенсивности)
– Регистрация: https://anthropic.com

### Telegram + BotFather

– Аккаунт сам по себе бесплатный
– Через @BotFather создаёшь по одному боту на каждого агента (`/newbot` → имя → токен)
– Токены сохраняй в `~/.secrets/` (или твой эквивалент), никогда не в git

### VPS provider

Нужен для self-host gbrain. Рекомендация по соотношению цена / производительность:

– Hetzner CPX42 (4 vCPU / 8 GiB / 160 GB SSD): ~12€/мес – проверенная конфигурация
– Альтернативы: DigitalOcean, Vultr, Linode – любой провайдер с Ubuntu 22.04+

## Опционально

### Domain для gbrain HTTPS

– Купить можно через любого регистратора (Cloudflare, Porkbun, REG.RU)
– Cloudflare DNS бесплатный – удобно для управления и quick switch
– Без domain можно гонять gbrain на чистом IP, но Telegram webhook требует HTTPS, и self-signed certs Telegram не примет

### SSL сертификат

– Let's Encrypt бесплатно
– Caddy v2 автоматически получает и обновляет (рекомендую вместо ручной возни с certbot)
– `apt install caddy` → один Caddyfile с reverse proxy на твой gbrain port

## Чек-лист «всё установлено?»

Прогони эти команды – все должны вернуть версию без ошибок:

```bash
claude --version
gh --version && gh auth status
git --version
node --version    # v22+
python3 --version # 3.12+
bun --version
```

Дополнительно проверь:

– GitHub SSH работает: `ssh -T git@github.com` → видишь приветствие со своим username
– Anthropic Max активен: открой `claude` локально, отправь любой запрос – ответ должен прийти без сообщений о биллинге
– У тебя есть хотя бы один Telegram bot token (можно проверить вручную через curl `https://api.telegram.org/bot<TOKEN>/getMe`)
– VPS доступен по SSH: `ssh root@<your-ip>` отвечает за < 1 сек

Если все шесть пунктов зелёные – можешь открывать `lessons/lesson-3-agents-with-gbrain/` и начинать.
