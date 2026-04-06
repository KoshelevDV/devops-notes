# ClawFeed — AI-powered новостной дайджест

**GitHub:** https://github.com/kevinho/clawfeed  
**Live demo:** https://clawfeed.kevinhe.io  
**Лицензия:** MIT  
**Дата заметки:** 2026-03-29

---

## Что это

AI-powered инструмент для сбора и структурирования новостных дайджестов из множества источников. Генерирует дайджесты с частотой 4h/daily/weekly/monthly.

**Поддерживаемые источники:**
| Тип | Пример |
|---|---|
| `rss` | любой RSS/Atom URL |
| `reddit` | `/r/devops` (через RSS: `reddit.com/r/devops/hot.rss`) |
| `hackernews` | топ HN |
| `github_trending` | по языку/периоду |
| `twitter_feed` | `@username` |
| `website` | scraping любого URL |
| `digest_feed` | другой ClawFeed пользователь |
| `custom_api` | JSON endpoint |

> ⚠️ Reddit JSON API (`/r/xxx.json`) блокируется по IP из-за рубежа. Использовать **RSS** (`/r/xxx/hot.rss`) — работает без ограничений.

---

## Архитектура с AI-агентом

```
ClawFeed сервер :8767
  └── SQLite БД (хранит дайджесты)
  └── Web UI (читать историю дайджестов)
  └── RSS/JSON Feed экспорт

AI-агент (Hermes, генератор)
  └── cron каждый день 09:00
      ├── собираю RSS/HN/GitHub через web_fetch
      ├── суммаризирую своим интеллектом
      ├── POST /api/digests → ClawFeed (сохраняю)
      └── отправляю сводку в Telegram
```

**ClawFeed = витрина.** Агент = движок генерации. Отдельный LLM не нужен.

---

## Установка

### Как OpenClaw skill (рекомендуется)

```bash
# Вариант 1 — через clawhub (если появится)
clawhub install clawfeed

# Вариант 2 — из GitHub напрямую
cd ~/.openclaw/workspace/skills/
git clone --depth=1 https://github.com/kevinho/clawfeed.git
```

OpenClaw автоматически подхватывает `SKILL.md` из папки skills.

### Зависимости

```bash
cd ~/.openclaw/workspace/skills/clawfeed
npm install
```

Требует: Node.js 18+, SQLite (через `better-sqlite3`, bundled).

---

## Конфигурация

### .env файл

```bash
cp .env.example .env
```

```env
# Google OAuth (опционально — без него read-only режим)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
SESSION_SECRET=<random 32 hex>
OAUTH_STATE_SECRET=

# API Key для создания дайджестов (обязателен для POST /api/digests)
API_KEY=clawfeed-local-key

# Server
DIGEST_PORT=8767

# CORS
ALLOWED_ORIGINS=localhost,127.0.0.1,10.0.30.18
```

Сгенерировать SESSION_SECRET:
```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
```

---

## Запуск

```bash
# Запуск сервера
cd ~/.openclaw/workspace/skills/clawfeed
node --env-file=.env src/server.mjs

# Или через npm
npm start

# Проверка
curl http://localhost:8767/api/digests
```

Web UI: `http://<IP>:8767`

---

## Настройка источников (через БД напрямую)

Если нет Google OAuth — создать системного юзера и источники через Node.js:

```bash
node -e "
import('./src/db.mjs').then(({ getDb, createSource, upsertUser }) => {
  const db = getDb('./data/digest.db');
  const user = upsertUser(db, { googleId: 'local-admin', email: 'admin@local', name: 'Admin', avatar: '' });
  
  const sources = [
    { name: 'r/selfhosted', type: 'reddit', config: { subreddit: 'selfhosted', sort: 'hot', limit: 20 } },
    { name: 'r/homelab', type: 'reddit', config: { subreddit: 'homelab', sort: 'hot', limit: 15 } },
    { name: 'r/kubernetes', type: 'reddit', config: { subreddit: 'kubernetes', sort: 'hot', limit: 15 } },
    { name: 'r/devops', type: 'reddit', config: { subreddit: 'devops', sort: 'hot', limit: 15 } },
    { name: 'HackerNews', type: 'hackernews', config: {} },
    { name: 'GitHub Trending', type: 'github_trending', config: { language: '', since: 'daily' } },
  ];
  
  for (const s of sources) {
    const r = createSource(db, { ...s, config: JSON.stringify(s.config), createdBy: user.id, isPublic: true });
    console.log('Created:', s.name, '->', r.id);
  }
});
"
```

---

## API

### Создать дайджест

```bash
curl -s -X POST http://localhost:8767/api/digests \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer clawfeed-local-key' \
  -d '{
    "type": "daily",
    "title": "Tech Digest — 2026-03-29",
    "content": "# Дайджест\n\n...",
    "items": []
  }'
# → {"id": 1}
```

**Типы дайджестов:** `4h` | `daily` | `weekly` | `monthly`

### Получить дайджесты

```bash
curl http://localhost:8767/api/digests?type=daily&limit=10
```

### RSS/JSON Feed экспорт

```
http://localhost:8767/feed/<slug>.rss
http://localhost:8767/feed/<slug>.json
```

---

## Интеграция с OpenClaw — cron дайджест

### Cron задача (настроена, каждый день 09:00 Саратов)

ID: `5d64f312-9def-4129-8fa5-3f438e5ac767`

**Источники в дайджесте:**
- Reddit RSS: r/selfhosted, r/homelab, r/kubernetes, r/cybersecurity, r/devops, r/golang, r/MachineLearning, r/artificial
- GitHub Trending (все языки, за день)
- HackerNews топ-8
- Release Notes: GitLab, Kubernetes, ELMA365, OpenClaw, Rancher, Helm, Nexus
- Security: CNCF blog, CVE

**Формат:** топ-10 пунктов на русском → Telegram + ClawFeed

### Проверить/изменить cron

```
# Через OpenClaw cron tool:
action: list  # посмотреть все задачи
action: run, jobId: 5d64f312-...  # запустить вручную
action: update, jobId: ..., patch: {enabled: false}  # выключить
```

---

## Структура файлов

```
skills/clawfeed/
├── src/
│   ├── server.mjs      # API сервер (Fastify)
│   └── db.mjs          # SQLite операции
├── web/
│   └── index.html      # SPA фронтенд
├── templates/
│   ├── curation-rules.md   # Правила фильтрации контента
│   └── digest-prompt.md    # Промпт для AI суммаризации
├── data/
│   └── digest.db       # SQLite база (создаётся автоматически)
├── .env                # Конфиг (создать из .env.example)
└── SKILL.md            # OpenClaw skill descriptor
```

---

## Кастомизация

### Правила фильтрации

`templates/curation-rules.md` — редактировать чтобы изменить что включать/исключать из дайджестов.

### Формат дайджеста

`templates/digest-prompt.md` — промпт для AI суммаризации (если используется встроенный AI, не агент).

---

## Reverse Proxy (nginx)

```nginx
location /clawfeed/api/ {
    rewrite ^/clawfeed/api/(.*) /$1 break;
    proxy_pass http://127.0.0.1:8767;
}
location /clawfeed/ {
    alias /home/user/.openclaw/workspace/skills/clawfeed/web/;
    try_files $uri $uri/ /clawfeed/index.html;
}
```

---

## Текущий инстанс

- **URL:** http://10.0.30.18:8767
- **Путь:** `/home/user/.openclaw/workspace/skills/clawfeed/`
- **API Key:** `clawfeed-local-key`
- **БД:** `data/digest.db`
- **Процесс:** запущен вручную (нет systemd сервиса)

> TODO: добавить systemd unit или docker-compose для автозапуска

---

## Ссылки

- GitHub: https://github.com/kevinho/clawfeed
- ClawHub: https://clawhub.ai/skills/clawfeed
- Документация: см. README.md в репо
