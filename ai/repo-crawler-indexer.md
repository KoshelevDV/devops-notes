# Краулер-индексатор репозиториев GitHub/GitLab

> Цель: автоматически собирать, классифицировать и анализировать репозитории
> по направлениям (Kubernetes-аддоны, Yandex-утилиты, DevSecOps-инструменты и т.д.)

---

## Архитектура системы

```
┌─────────────────────────────────────────────────────────────────┐
│                         CRAWLER                                 │
│                                                                 │
│  ┌────────────┐    ┌─────────────┐    ┌──────────────────────┐ │
│  │  Scheduler │───►│  Fetcher    │───►│  LLM Classifier      │ │
│  │            │    │             │    │                      │ │
│  │ - new repos│    │ GitHub API  │    │ - определить направл.│ │
│  │ - re-index │    │ GitLab API  │    │ - извлечь теги       │ │
│  │ - watch    │    │ README fetch│    │ - оценить качество   │ │
│  └────────────┘    └─────────────┘    └──────────────────────┘ │
│                                                ▼                │
│                                    ┌──────────────────────┐    │
│                                    │       Database       │    │
│                                    │  (SQLite / Postgres) │    │
│                                    │                      │    │
│                                    │ repos, tags,         │    │
│                                    │ directions, stats    │    │
│                                    └──────────────────────┘    │
│                                                ▼                │
│                                    ┌──────────────────────┐    │
│                                    │   Analytics / API    │    │
│                                    │                      │    │
│                                    │ - статистика         │    │
│                                    │ - топ по направлению │    │
│                                    │ - тренды             │    │
│                                    └──────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Источники данных

### GitHub API (REST + GraphQL)

```python
import httpx

GH_TOKEN = "ghp_..."
HEADERS = {"Authorization": f"Bearer {GH_TOKEN}", "Accept": "application/vnd.github.v3+json"}

# 1. Поиск по запросу
def search_repos(query: str, page=1, per_page=100):
    r = httpx.get(
        "https://api.github.com/search/repositories",
        params={"q": query, "sort": "stars", "order": "desc",
                "per_page": per_page, "page": page},
        headers=HEADERS
    )
    return r.json()["items"]

# Примеры запросов
queries = [
    "topic:kubernetes language:go",
    "topic:yandex-cloud stars:>10",
    "kubectl plugin in:description",
    "yandex in:description,name stars:>5 language:python",
    "helm chart in:description pushed:>2024-01-01",
]

# 2. Топики репозитория (теги)
def get_repo_topics(owner, repo):
    r = httpx.get(
        f"https://api.github.com/repos/{owner}/{repo}/topics",
        headers={**HEADERS, "Accept": "application/vnd.github.mercy-preview+json"}
    )
    return r.json().get("names", [])

# 3. GraphQL — более эффективно (1 запрос вместо N)
GQL_QUERY = """
query($query: String!, $after: String) {
  search(query: $query, type: REPOSITORY, first: 100, after: $after) {
    pageInfo { hasNextPage endCursor }
    nodes {
      ... on Repository {
        nameWithOwner
        description
        stargazerCount
        forkCount
        primaryLanguage { name }
        repositoryTopics(first: 10) {
          nodes { topic { name } }
        }
        pushedAt
        licenseInfo { name }
        isArchived
        openIssuesCount: issues(states: OPEN) { totalCount }
      }
    }
  }
}
"""

def graphql_search(query: str):
    r = httpx.post(
        "https://api.github.com/graphql",
        json={"query": GQL_QUERY, "variables": {"query": query}},
        headers=HEADERS
    )
    return r.json()["data"]["search"]["nodes"]
```

### GitLab API (self-hosted)

```python
GL_TOKEN = "glpat-..."
GL_URL = "https://gitlab.fgdevelop.tech"

def search_gl_projects(search: str, page=1):
    r = httpx.get(
        f"{GL_URL}/api/v4/projects",
        params={"search": search, "order_by": "star_count",
                "sort": "desc", "per_page": 100, "page": page},
        headers={"PRIVATE-TOKEN": GL_TOKEN}
    )
    return r.json()

def get_gl_project(project_id: int):
    r = httpx.get(
        f"{GL_URL}/api/v4/projects/{project_id}",
        params={"statistics": True},
        headers={"PRIVATE-TOKEN": GL_TOKEN}
    )
    return r.json()
```

---

## База данных

```sql
-- SQLite / PostgreSQL schema

CREATE TABLE repos (
    id          TEXT PRIMARY KEY,  -- "github:owner/repo" или "gitlab:42"
    source      TEXT NOT NULL,     -- github / gitlab
    full_name   TEXT NOT NULL,
    url         TEXT NOT NULL,
    description TEXT,
    language    TEXT,
    stars       INTEGER DEFAULT 0,
    forks       INTEGER DEFAULT 0,
    open_issues INTEGER DEFAULT 0,
    license     TEXT,
    is_archived BOOLEAN DEFAULT FALSE,
    pushed_at   TIMESTAMP,
    indexed_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE repo_topics (
    repo_id TEXT REFERENCES repos(id),
    topic   TEXT NOT NULL,
    PRIMARY KEY (repo_id, topic)
);

-- LLM-классификация по направлениям
CREATE TABLE repo_directions (
    repo_id     TEXT REFERENCES repos(id),
    direction   TEXT NOT NULL,     -- "kubernetes-addon", "yandex-cloud", etc.
    confidence  REAL,              -- 0.0 - 1.0
    reason      TEXT,              -- почему LLM так решил
    classified_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (repo_id, direction)
);

-- Справочник направлений
CREATE TABLE directions (
    slug        TEXT PRIMARY KEY,
    name        TEXT NOT NULL,
    description TEXT,
    keywords    TEXT   -- JSON array для быстрой фильтрации
);

-- Снепшоты для трендов (еженедельно)
CREATE TABLE repo_snapshots (
    repo_id   TEXT REFERENCES repos(id),
    week      DATE NOT NULL,
    stars     INTEGER,
    forks     INTEGER,
    PRIMARY KEY (repo_id, week)
);
```

```python
# Предзаполнение справочника направлений
DIRECTIONS = [
    {
        "slug": "kubernetes-addon",
        "name": "Kubernetes аддоны и операторы",
        "keywords": ["kubernetes", "k8s", "kubectl", "operator", "helm", "crd"],
        "description": "Операторы, плагины kubectl, Helm charts, CRD-контроллеры"
    },
    {
        "slug": "yandex-cloud",
        "name": "Yandex Cloud утилиты",
        "keywords": ["yandex", "yc", "yandex-cloud", "ydb", "yandex-mdb"],
        "description": "SDK, CLI, terraform-провайдеры, интеграции с YC"
    },
    {
        "slug": "devsecops",
        "name": "DevSecOps инструменты",
        "keywords": ["sast", "dast", "sca", "security", "vulnerability", "trivy", "semgrep"],
        "description": "Сканеры, SIEM, SOAR, AppSec-платформы"
    },
    {
        "slug": "observability",
        "name": "Observability / мониторинг",
        "keywords": ["prometheus", "grafana", "opentelemetry", "tracing", "logging", "loki"],
        "description": "Метрики, трейсинг, логирование, алертинг"
    },
    {
        "slug": "cicd",
        "name": "CI/CD инструменты",
        "keywords": ["pipeline", "cicd", "gitlab-ci", "github-actions", "tekton", "argocd"],
        "description": "Системы доставки, GitOps, деплой-инструменты"
    },
    {
        "slug": "infra-as-code",
        "name": "Infrastructure as Code",
        "keywords": ["terraform", "ansible", "pulumi", "crossplane", "cdktf"],
        "description": "Terraform-модули, Ansible роли, провайдеры"
    },
]
```

---

## LLM-классификатор

Ключевая часть — LLM определяет направление когда простого keyword-match недостаточно.

```python
CLASSIFY_PROMPT = """
Ты классифицируешь open-source репозитории по направлениям.

Доступные направления:
{directions}

Репозиторий:
- Название: {name}
- Описание: {description}
- Язык: {language}
- Топики: {topics}
- README (первые 500 символов): {readme_preview}

Определи к каким направлениям относится репозиторий.
Репозиторий может относиться к нескольким направлениям.

Ответь строго JSON:
{{
  "directions": [
    {{"slug": "kubernetes-addon", "confidence": 0.95, "reason": "kubectl plugin для управления namespace"}}
  ],
  "quality_score": 0.8,
  "quality_reason": "активно поддерживается, хорошая документация, 500+ звёзд",
  "summary": "краткое описание что делает проект (1-2 предложения)"
}}
"""

def classify_repo(repo: dict, directions: list, readme: str = "") -> dict:
    from glm_client import llm  # ваш клиент

    directions_text = "\n".join([
        f"- {d['slug']}: {d['description']} (ключевые слова: {', '.join(d['keywords'])})"
        for d in directions
    ])

    prompt = CLASSIFY_PROMPT.format(
        directions=directions_text,
        name=repo["full_name"],
        description=repo.get("description", ""),
        language=repo.get("language", ""),
        topics=", ".join(repo.get("topics", [])),
        readme_preview=readme[:500] if readme else "недоступен"
    )

    response = llm(
        system="Ты эксперт по open-source экосистеме. Отвечай только JSON.",
        user=prompt
    )

    try:
        return json.loads(response)
    except json.JSONDecodeError:
        return {"directions": [], "quality_score": 0, "summary": ""}
```

---

## Agentic loop краулера

```python
import asyncio
import httpx
import json
from datetime import datetime, timedelta

class RepoCrawler:
    def __init__(self, db, llm, directions):
        self.db = db
        self.llm = llm
        self.directions = directions
        self.stats = {"fetched": 0, "classified": 0, "skipped": 0}

    async def crawl_query(self, query: str, source: str = "github"):
        """Обойти все страницы по запросу"""
        page = 1
        while True:
            repos = await self.fetch_page(query, source, page)
            if not repos:
                break

            for repo in repos:
                if self.db.exists(repo["id"]):
                    # Обновить только если давно не индексировали
                    if self.needs_reindex(repo["id"]):
                        await self.index_repo(repo)
                    else:
                        self.stats["skipped"] += 1
                else:
                    await self.index_repo(repo)

            page += 1
            await asyncio.sleep(1)  # rate limiting

    async def index_repo(self, repo: dict):
        """Индексировать один репозиторий"""
        # 1. Получить README
        readme = await self.fetch_readme(repo)

        # 2. Классифицировать через LLM
        classification = self.llm.classify(repo, self.directions, readme)

        # 3. Сохранить
        self.db.upsert_repo(repo)
        self.db.set_directions(repo["id"], classification["directions"])
        self.db.set_summary(repo["id"], classification.get("summary", ""))

        self.stats["classified"] += 1

    def needs_reindex(self, repo_id: str) -> bool:
        """Переиндексировать если больше недели не обновляли"""
        last = self.db.get_indexed_at(repo_id)
        return last is None or (datetime.now() - last) > timedelta(days=7)

async def main():
    crawler = RepoCrawler(db=Database(), llm=LLMClassifier(), directions=DIRECTIONS)

    # Запросы для каждого направления
    crawl_plan = [
        ("topic:kubernetes-operator language:go stars:>50", "github"),
        ("topic:helm-chart pushed:>2024-01-01", "github"),
        ("yandex-cloud SDK in:description stars:>10", "github"),
        ("topic:prometheus-exporter", "github"),
        ("terraform-provider in:name stars:>20", "github"),
    ]

    for query, source in crawl_plan:
        print(f"Crawling: {query}")
        await crawler.crawl_query(query, source)

    print(f"Done: {crawler.stats}")
```

---

## Аналитика и статистика

```python
# Топ репозиториев по направлению
def top_by_direction(direction_slug: str, limit: int = 20):
    return db.query("""
        SELECT r.full_name, r.description, r.stars, r.language,
               r.pushed_at, rd.confidence, rd.reason
        FROM repos r
        JOIN repo_directions rd ON r.id = rd.repo_id
        WHERE rd.direction = ? AND r.is_archived = FALSE
        ORDER BY r.stars DESC
        LIMIT ?
    """, [direction_slug, limit])

# Статистика по языкам в направлении
def language_stats(direction_slug: str):
    return db.query("""
        SELECT r.language, COUNT(*) as count, AVG(r.stars) as avg_stars
        FROM repos r
        JOIN repo_directions rd ON r.id = rd.repo_id
        WHERE rd.direction = ?
        GROUP BY r.language
        ORDER BY count DESC
    """, [direction_slug])

# Активность (новые/обновлённые за последний месяц)
def activity_stats():
    return db.query("""
        SELECT rd.direction, COUNT(*) as total,
               SUM(CASE WHEN r.pushed_at > date('now', '-30 days') THEN 1 ELSE 0 END) as active_month
        FROM repos r
        JOIN repo_directions rd ON r.id = rd.repo_id
        GROUP BY rd.direction
        ORDER BY total DESC
    """)

# Тренды — рост звёзд за неделю
def trending(direction_slug: str):
    return db.query("""
        SELECT r.full_name, r.stars,
               r.stars - s.stars as stars_growth
        FROM repos r
        JOIN repo_snapshots s ON r.id = s.repo_id
        JOIN repo_directions rd ON r.id = rd.repo_id
        WHERE rd.direction = ?
          AND s.week = date('now', '-7 days', 'weekday 1')
        ORDER BY stars_growth DESC
        LIMIT 10
    """, [direction_slug])
```

---

## Rate limits — важно

| Источник | Лимит | Как обойти |
|---|---|---|
| GitHub REST API (с токеном) | 5000 req/час | 1 req/сек + кеш |
| GitHub GraphQL API | 5000 points/час | Батчить в 1 запрос |
| GitHub Search API | 30 req/мин | `asyncio.sleep(2)` между страницами |
| GitLab self-hosted | Зависит от конфига | Обычно без лимитов |

```python
# Умный rate limiter
import time

class RateLimiter:
    def __init__(self, calls_per_second=1):
        self.calls_per_second = calls_per_second
        self.last_call = 0

    async def wait(self):
        now = time.time()
        elapsed = now - self.last_call
        if elapsed < 1 / self.calls_per_second:
            await asyncio.sleep(1 / self.calls_per_second - elapsed)
        self.last_call = time.time()
```

---

## Выходной формат

### JSON API (FastAPI)
```python
@app.get("/directions/{slug}")
def get_direction(slug: str, sort: str = "stars", limit: int = 50):
    return {
        "direction": directions[slug],
        "repos": top_by_direction(slug, limit),
        "stats": {
            "total": count_by_direction(slug),
            "languages": language_stats(slug),
            "activity": activity_stats_for(slug)
        }
    }
```

### Markdown отчёт (в devops-notes)
```python
def generate_report(direction_slug: str) -> str:
    repos = top_by_direction(direction_slug, 20)
    stats = language_stats(direction_slug)

    md = f"# {directions[direction_slug]['name']}\n\n"
    md += f"Всего проектов: {count_by_direction(direction_slug)}\n\n"
    md += "## Топ проектов\n\n"
    for r in repos:
        md += f"- [{r['full_name']}]({r['url']}) ⭐{r['stars']} — {r['description']}\n"
    return md
```

---

## Практический план запуска

1. **День 1** — схема БД + GitHub fetcher + 3-4 запроса для одного направления (kubernetes)
2. **День 2** — LLM-классификатор + загрузить 500-1000 репо
3. **День 3** — аналитические запросы + markdown-отчёт
4. **Далее** — расширять направления, добавить GitLab, cron-обновление

```bash
# Запуск как cron-задача (еженедельно)
# В OpenClaw cron:
# schedule: every week
# payload: python /opt/projects/repo-crawler/main.py
```

---

## Ссылки

- GitHub REST API Search: https://docs.github.com/en/rest/search
- GitHub GraphQL API: https://docs.github.com/en/graphql
- GitLab Projects API: https://docs.gitlab.com/ee/api/projects.html
- GitHub Topics: https://github.com/topics
