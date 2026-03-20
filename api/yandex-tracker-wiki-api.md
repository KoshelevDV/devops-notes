# Яндекс Трекер и Вики — API справочник

> Для проекта `ya-wiki-agent`: https://github.com/KoshelevDV/ya-wiki-agent

---

## Аутентификация (общее для обоих API)

**Организация Яндекс 360 для бизнеса → OAuth 2.0**

### Получить OAuth токен
1. Идти на https://oauth.yandex.ru/
2. Создать приложение → "Для доступа к API или отладки"
3. Добавить разрешения:
   - Трекер: `tracker:read` (только чтение) или `tracker:write` (всё)
   - Вики: `wiki:read` (только чтение) или `wiki:write` (всё)
4. После создания: `https://oauth.yandex.ru/authorize?response_type=token&client_id=<ClientID>`
5. Скопировать токен из URL после авторизации

### Узнать Org ID
Трекер → Администрирование → Организации → скопировать значение "идентификатор"

### Проверить доступ
```bash
# Трекер
curl -X GET 'https://api.tracker.yandex.net/v3/myself' \
  -H 'Authorization: OAuth <token>' \
  -H 'X-Org-ID: <org_id>'

# Вики — получить страницу
curl -X GET 'https://api.wiki.yandex.net/v1/pages?slug=mypage' \
  -H 'Authorization: OAuth <token>' \
  -H 'X-Org-Id: <org_id>'
```

---

## Яндекс Трекер API

**Base URL:** `https://api.tracker.yandex.net/v3`
**Docs:** https://yandex.ru/support/tracker/ru/api-ref/about-api

### Обязательные заголовки
```
Authorization: OAuth <token>
X-Org-ID: <org_id>           # для Яндекс 360
# или
X-Cloud-Org-ID: <cloud_org_id>  # для Yandex Cloud
```

### Пагинация
- По умолчанию 50 записей на страницу
- Параметры: `?page=1&perPage=50`
- Заголовки ответа: `X-Total-Pages`, `X-Total-Count`

---

### Задачи (Issues)

#### Получить задачу
```
GET /v3/issues/{issueKey}
```
```bash
curl 'https://api.tracker.yandex.net/v3/issues/AI_SLOP-1' \
  -H 'Authorization: OAuth <token>' \
  -H 'X-Org-ID: <org_id>'
```

**Ответ:**
```json
{
  "id": "...",
  "key": "AI_SLOP-1",
  "summary": "Название задачи",
  "description": "Описание",
  "status": { "key": "open", "display": "Открыт" },
  "type": { "key": "task", "display": "Задача" },
  "priority": { "key": "normal", "display": "Средний" },
  "assignee": { "display": "Имя Фамилия" },
  "createdAt": "2026-03-20T...",
  "updatedAt": "2026-03-20T...",
  "queue": { "key": "AI_SLOP" },
  "tags": ["..."]
}
```

#### Поиск задач
```
POST /v3/issues/_search
```
```json
{
  "filter": {
    "queue": "AI_SLOP",
    "status": "open"
  },
  "order": "-updatedAt"
}
```
Или через язык запросов:
```json
{
  "query": "Queue: AI_SLOP Status: open \"Sort by\": Updated DESC"
}
```

**Фильтрация по очереди (относительная пагинация):**
```json
{ "queue": "AI_SLOP" }
```

**Параметры фильтра:**
- `queue` — ключ очереди
- `status` — статус: `open`, `inProgress`, `resolved`, `closed`
- `assignee` — исполнитель (uid или `empty()`)
- `type` — тип: `task`, `bug`, `epic`, `story`
- `priority` — приоритет: `critical`, `blocker`, `major`, `normal`, `minor`, `trivial`
- `tags` — теги

**expand параметры:**
- `?expand=transitions` — переходы по ЖЦ
- `?expand=attachments` — вложения
- `?expand=comments` — комментарии

#### Создать задачу
```
POST /v3/issues
```
```json
{
  "summary": "Название задачи",
  "queue": { "key": "AI_SLOP" },
  "type": { "key": "task" },
  "priority": { "key": "normal" },
  "description": "Описание в **Markdown**",
  "tags": ["light", "infra"],
  "parent": { "key": "AI_SLOP-1" }
}
```

**Типы задач:** `task`, `bug`, `epic`, `story`, `improvement`
**Приоритеты:** `blocker`, `critical`, `major`, `normal`, `minor`, `trivial`
**Описание:** Yandex Flavored Markdown, переносы через `\n`

#### Редактировать задачу (PATCH)
```
PATCH /v3/issues/{issueKey}
```
```json
{
  "summary": "Новое название",
  "description": "Новое описание",
  "status": { "key": "inProgress" },
  "tags": { "add": ["new-tag"] }
}
```

---

### Очереди (Queues)

#### Получить очередь
```
GET /v3/queues/{queueKey}
```

#### Список очередей
```
GET /v3/queues
```

**Ответ включает:** name, description, lead, defaultType, defaultPriority, issueTypes, workflows

---

### Создать очередь (Queue)
```
POST /v3/queues
```
```json
{
  "key": "AI_SLOP",
  "name": "AI Assistant Project",
  "lead": { "id": "<user_id>" },
  "defaultType": { "key": "task" },
  "defaultPriority": { "key": "normal" }
}
```

---

## Яндекс Вики API

**Base URL:** `https://api.wiki.yandex.net/v1`
**Docs:** https://yandex.ru/support/wiki/ru/api-ref/about

### Обязательные заголовки
```
Authorization: OAuth <token>
X-Org-Id: <org_id>           # для Яндекс 360 (с маленькой d!)
# или
X-Cloud-Org-Id: <cloud_org_id>
```
> ⚠️ Заголовок для Вики: `X-Org-Id` (не `X-Org-ID` как в Трекере!)

### Ограничения
- Нельзя использовать сервисный аккаунт — только пользовательский OAuth токен
- Права API = права пользователя в Вики

---

### Страницы (Pages)

#### Получить страницу по slug
```
GET /v1/pages?slug={slug}
```
```bash
curl 'https://api.wiki.yandex.net/v1/pages?slug=ai_wiki/architecture' \
  -H 'Authorization: OAuth <token>' \
  -H 'X-Org-Id: <org_id>'
```

#### Получить содержимое страницы
```
GET /v1/pages/{pageId}/body
```
или
```
GET /v1/pages?slug={slug}&expand=body
```

#### Список дочерних страниц
```
GET /v1/pages?parent={pageId}
```

#### Поиск по страницам
```
GET /v1/pages?q={query}
```

#### Создать страницу (только в разрешённых разделах)
```
POST /v1/pages
```
```json
{
  "title": "Название страницы",
  "slug": "ai_wiki/new-page",
  "body": "# Заголовок\n\nТекст страницы в Markdown",
  "parentId": "<parent_page_id>"
}
```

#### Обновить страницу
```
PATCH /v1/pages/{pageId}
```
```json
{
  "body": "Новый контент",
  "title": "Новый заголовок"
}
```

---

## Python клиенты

### yandex_tracker_client (официальный)
```bash
pip install yandex_tracker_client
```
```python
from yandex_tracker_client import TrackerClient

client = TrackerClient(
    token='y0_AgAAAA...',
    org_id='12345678'  # Яндекс 360
)

# Получить задачу
issue = client.issues['AI_SLOP-1']
print(issue.summary, issue.status)

# Создать задачу
new_issue = client.issues.create(
    queue='AI_SLOP',
    summary='Название',
    type={'key': 'task'},
)

# Поиск
issues = client.issues.find(filter={'queue': 'AI_SLOP', 'status': 'open'})
```

### httpx (для Вики API — нет официального клиента)
```python
import httpx

class YandexWikiClient:
    BASE_URL = "https://api.wiki.yandex.net/v1"
    
    def __init__(self, token: str, org_id: str):
        self.headers = {
            "Authorization": f"OAuth {token}",
            "X-Org-Id": org_id,
        }
    
    def get_page(self, slug: str) -> dict:
        r = httpx.get(f"{self.BASE_URL}/pages", 
                      params={"slug": slug}, 
                      headers=self.headers)
        r.raise_for_status()
        return r.json()
    
    def search(self, query: str) -> list:
        r = httpx.get(f"{self.BASE_URL}/pages",
                      params={"q": query},
                      headers=self.headers)
        r.raise_for_status()
        return r.json()
    
    def get_body(self, page_id: str) -> str:
        r = httpx.get(f"{self.BASE_URL}/pages/{page_id}/body",
                      headers=self.headers)
        r.raise_for_status()
        return r.json().get("body", "")
    
    def create_page(self, title: str, slug: str, body: str, parent_id: str = None) -> dict:
        payload = {"title": title, "slug": slug, "body": body}
        if parent_id:
            payload["parentId"] = parent_id
        r = httpx.post(f"{self.BASE_URL}/pages",
                       json=payload,
                       headers=self.headers)
        r.raise_for_status()
        return r.json()
```

---

## Важные нюансы

| Момент | Детали |
|---|---|
| X-Org-ID vs X-Org-Id | Трекер: `X-Org-ID`, Вики: `X-Org-Id` (разный регистр!) |
| Пагинация | По умолчанию 50 записей; использовать `page` + `perPage` |
| Markdown | Yandex Flavored Markdown, переносы через `\n` |
| Часовой пояс | API всегда возвращает UTC±00:00 |
| Права | Токен = права пользователя; нет прав → 403 |
| Сервисные аккаунты | Трекер: через IAM-токен; Вики: НЕЛЬЗЯ, только пользователь |
| Версия API | Трекер: v3 (актуальная); Вики: v1 |

---

## Эндпоинты для ya-wiki-agent MCP сервера

### Только чтение (основной режим)
```
GET  /v3/issues/_search         → поиск задач в AI_SLOP
GET  /v3/issues/{key}           → получить задачу
GET  /v3/queues/{key}           → инфо об очереди
GET  /v1/pages?slug={slug}      → получить страницу Вики по slug
GET  /v1/pages?q={query}        → поиск по Вики
GET  /v1/pages/{id}/body        → содержимое страницы
```

### Запись (только в разрешённых местах)
```
POST /v3/issues                 → создать задачу в AI_SLOP
POST /v1/pages                  → создать страницу в AI_WIKI
PATCH /v1/pages/{id}            → обновить страницу в AI_WIKI
```
