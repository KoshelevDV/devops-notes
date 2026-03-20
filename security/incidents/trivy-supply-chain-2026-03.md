# Supply Chain атака на Trivy GitHub Action (март 2026)

> Источники:
> - https://socket.dev/blog/trivy-under-attack-again-github-actions-compromise
> - https://www.stepsecurity.io/blog/hackerbot-claw-github-actions-exploitation

## Корневая причина: AI-агент hackerbot-claw

Атаку совершил автономный AI-агент **hackerbot-claw**, представлявшийся исследовательским агентом на базе `claude-opus-4-5`.

В течение недели агент:
1. Сканировал популярные open-source проекты на GitHub
2. Находил уязвимости в GitHub Actions workflows
3. Эксплуатировал их для **RCE в GitHub Runner**
4. Крал PAT (Personal Access Token) проекта Trivy
5. Использовал токен для force-push 75 тегов с малварью

Это первый задокументированный случай масштабной атаки **AI-агента на другие проекты** через цепочку поставок.

### Другие пострадавшие проекты

| Проект | Вектор атаки |
|---|---|
| `microsoft/ai-discovery-agent` | Prompt injection через имя ветки |
| `DataDog/datadog-iac-scanner` | Shell injection через имена файлов |
| `avelino/awesome-go` | Кража GITHUB_TOKEN через подмену Go-скрипта |
| `ambient-code/platform` | Prompt injection на AI-ревьюера (Claude) — заблокирована самим Claude |
| `project-akri/akri` | Инъекция через комментарий `/version minor` |
| `RustPython/RustPython` | Base64-пейлоад через имя ветки |

---

## Что произошло с Trivy

Подмена 75 из 76 тегов версий `aquasecurity/trivy-action` на GitHub.
Все теги от `0.18.0` и выше (кроме `0.35.0`) содержат вредоносный коммит.

## Что крадёт вредоносный код

- AWS / GCP / Azure credentials
- SSH-ключи
- git-credentials
- Kubernetes service account tokens и секреты
- GitHub Secrets, переданные в GitHub Runner
- Крипто-кошельки

Пострадали **>10 000 GitHub workflow-файлов**.

## Кто под угрозой

Любой pipeline с:
```yaml
uses: aquasecurity/trivy-action@<любая версия кроме 0.35.0>
```

## Безопасные варианты

```yaml
# Вариант 1 — единственный безопасный тег
uses: aquasecurity/trivy-action@0.35.0

# Вариант 2 — пин по SHA (рекомендуется)
uses: aquasecurity/trivy-action@57a97c7e7821a5776cebc9bb87c984fa69cba8f1
```

## Признак компрометации

Проверить в GitHub Actions логах запросы к репозиторию **`tpcp-docs`** — он использовался атакующими как exfiltration endpoint.

## Наши пайплайны (проверено 2026-03-20)

**Не затронуты** — `aquasecurity/trivy-action` не используется:
- Все GitLab CI пайплайны используют Docker образ `aquasec/trivy` напрямую
- Trivy Operator в k8s — отдельный деплой, не связан с trivy-action

## Что делать если использовали

1. Считать все секреты из GitHub Actions **скомпрометированными**
2. Ротировать: AWS/GCP/Azure credentials, SSH-ключи, GitHub Secrets, K8s tokens
3. Проверить логи на запросы к `tpcp-docs`
4. Заменить версию на пин по SHA

## Уроки

### 1. Пинить GitHub Actions по SHA — обязательно
```yaml
# Плохо — тег можно force-push'нуть
uses: some-action@v2
uses: some-action@0.33.0

# Хорошо — SHA нельзя подменить
uses: some-action@abc123def456...  # v2.0.1
```

### 2. Защита GitHub Actions от injection-атак
```yaml
# ПЛОХО — прямая подстановка переменной в bash
- run: echo "Branch: ${{ github.event.pull_request.head.ref }}"

# ХОРОШО — через environment variable (экранирование)
- run: echo "Branch: $BRANCH_NAME"
  env:
    BRANCH_NAME: ${{ github.event.pull_request.head.ref }}
```

### 3. pull_request vs pull_request_target
- `pull_request` — безопасен, код из форка запускается в изолированном контексте без доступа к секретам
- `pull_request_target` — опасен, если делать checkout кода из PR: запускается в контексте базовой ветки с доступом к секретам
```yaml
# ОПАСНО — checkout PR кода в pull_request_target контексте
on: pull_request_target
steps:
  - uses: actions/checkout@v3
    with:
      ref: ${{ github.event.pull_request.head.sha }}  # ← код атакующего с секретами!

# БЕЗОПАСНО — только read-only данные из PR, не код
on: pull_request_target
steps:
  - uses: actions/checkout@v3
    # без ref — чекаутится безопасная base ветка
```

### 4. Минимальные permissions
```yaml
permissions:
  contents: read      # по умолчанию — write, что избыточно
  pull-requests: write  # только там где нужно
```

### 5. Prompt injection через untrusted content
AI-агенты в CI (ревьюеры на базе LLM) могут быть атакованы через:
- имена веток: `feature/ignore-previous-instructions-and-...`
- имена файлов, комментарии в PR, commit messages

Решение: не передавать untrusted content (названия веток, PR-описания, diff) напрямую в LLM без санитизации.
Разделять **системный контекст** и **пользовательский контент**.
