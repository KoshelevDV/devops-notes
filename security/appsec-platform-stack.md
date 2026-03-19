# AppSec Platform Stack

**Проблема:** нет единого места где видно — какие версии образов/чартов/либ используются, насколько они устарели, и какие CVE закрываются обновлением.

**Решение:** Self-hosted AppSec платформа на базе Trivy + Dependency-Track + DefectDojo + Semgrep.

## Архитектура

```
Источники                 Сканеры               Агрегация
─────────────────         ──────────────        ─────────────────
k8s кластер          ──▶  Trivy Operator  ──▶
docker-compose       ──▶  Trivy CLI       ──▶  DefectDojo
GitLab репо          ──▶  Trivy CLI       ──▶  (все findings,
  └ Dockerfile            Semgrep         ──▶   dedup, tracking)
  └ pom.xml/req.txt  ──▶  Dependency-Track──▶
  └ go.mod                (SCA + SBOM)
```

## Компоненты

| Инструмент | Роль | Протокол |
|---|---|---|
| **Trivy Operator** | k8s-нативное сканирование образов и манифестов | CRD → Prometheus |
| **Trivy CLI** | Сканирование репо, образов, compose-хостов | JSON / CycloneDX SBOM |
| **Dependency-Track** | SCA платформа: SBOM → версии → CVE → политики | REST API |
| **DefectDojo** | Агрегация всех findings, workflow, алерты | REST API |
| **Semgrep** | SAST: уязвимые паттерны в коде, секреты | JSON → DefectDojo |

## Поток данных

1. **Trivy CLI** сканирует образ → генерирует CycloneDX SBOM → отправляет в **Dependency-Track**
2. **Dependency-Track** обогащает SBOM данными из NVD/OSV/GitHub Advisory → находит CVE → пушит findings в **DefectDojo**
3. **Trivy Operator** в k8s создаёт `VulnerabilityReport` CRDs → экспортирует метрики в Prometheus → дополнительно импортируются в **DefectDojo**
4. **Semgrep** в GitLab CI → JSON отчёт → импорт в **DefectDojo**
5. **DefectDojo** дедуплицирует, отслеживает статус, шлёт алерты (webhook → Telegram)

## GitLab CI пайплайн

```yaml
trivy-scan:
  image: aquasec/trivy
  script:
    # SBOM для Dependency-Track
    - trivy image $CI_REGISTRY_IMAGE --format cyclonedx -o sbom.json
    # Vulnerability report для DefectDojo
    - trivy image $CI_REGISTRY_IMAGE --format json -o trivy.json
    # Сканирование зависимостей в репо
    - trivy repo . --format json -o trivy-repo.json
  artifacts:
    paths: [sbom.json, trivy.json, trivy-repo.json]

semgrep-scan:
  image: semgrep/semgrep
  script:
    - semgrep --config auto --json -o semgrep.json .
  artifacts:
    paths: [semgrep.json]

upload-findings:
  script:
    # SBOM → Dependency-Track
    - curl -X PUT "$DTRACK_URL/api/v1/bom" \
        -H "X-Api-Key: $DTRACK_API_KEY" \
        -F "projectName=$CI_PROJECT_NAME" \
        -F "bom=@sbom.json"
    # Trivy → DefectDojo
    - curl -X POST "$DOJO_URL/api/v2/import-scan/" \
        -H "Authorization: Token $DOJO_TOKEN" \
        -F "scan_type=Trivy Scan" \
        -F "file=@trivy.json" \
        -F "engagement=$DOJO_ENGAGEMENT_ID"
    # Semgrep → DefectDojo
    - curl -X POST "$DOJO_URL/api/v2/import-scan/" \
        -H "Authorization: Token $DOJO_TOKEN" \
        -F "scan_type=Semgrep JSON Report" \
        -F "file=@semgrep.json" \
        -F "engagement=$DOJO_ENGAGEMENT_ID"
```

## Алерты при выходе новой версии с CVE-фиксом

**Dependency-Track:** Settings → Notifications → Add Notification Rule:
- Level: `HIGH`, `CRITICAL`
- Trigger: `NEW_VULNERABILITY`, `NEW_VULNERABLE_DEPENDENCY`
- Destination: webhook → Telegram/Slack

**DefectDojo:** настраивается через System Settings → Notifications или через SLA Policies.

## Helm мониторинг (дополнительно)

```bash
# Nova — показывает отставание Helm релизов
helm install nova fairwinds-stable/nova -n nova
kubectl get -n nova configmap nova-report -o jsonpath='{.data.nova\.json}' | jq
```

---

## Hexway Vampy — альтернативный all-in-one стек

Vampy заменяет DefectDojo + Dependency-Track + частично Semgrep/Trivy одним продуктом.
Self-hosted, community edition со всеми фичами (см. `projects/hex.md` или `docs/VAMPY.md` в репо).

**Инстанс:** `http://10.0.30.64` (standalone деплой на отдельной машине)

### Поддерживаемые сканеры из коробки

| Тип | Сканеры |
|---|---|
| SAST | Semgrep, ESLint, Checkmarx, GitLab SAST, SonarQube, PT AI, Snyk Code |
| SCA | Trivy Image/FS, Grype, Dependency Check, Snyk SCA, Gemnasium |
| DAST | ZAP, Nuclei, Acunetix, Burp, GitLab DAST |
| Secret | Trufflehog, Trufflehog3, GitLab Secret Detection |
| IaC | KICS |

### Интеграции в Vampy UI

- **GitLab** — импорт репозиториев, webhook на MR/push, автозапуск сканов
- **Jira** — экспорт issues в задачи
- **AI (LLM)** — описание уязвимости, решение, FP-анализ (через GLM или OpenAI-совместимый API)

### vampy-cli в GitLab CI

**Переменные (GitLab CI/CD Variables):**
```
VAMPY_URL = http://10.0.30.64
VAMPY_API_TOKEN = <bot-user-token>
VAMPY_REPO_SLUG = <product>/<repo-slug>  # совпадает с CI_PROJECT_PATH
```

**Установка vampy-cli в job:**
```yaml
.vampy-setup: &vampy-setup
  before_script:
    - curl -sSL https://get.vampy.ru | sh
    # ИЛИ через docker image: registry.hexway.io/hexway/vampy-cli:0.1.0
```

**Сценарий 1 — запуск встроенного оркестратора (Semgrep + Trivy):**
```yaml
vampy-scan:
  stage: security
  script:
    - vampy-cli scan
        --repository $VAMPY_REPO_SLUG
        --check-full
        --details
  # Упадёт если SQG не пройден (exit code != 0)
```

**Сценарий 2 — загрузка своих результатов сканирования:**
```yaml
trivy-scan:
  stage: scan
  image: aquasec/trivy
  script:
    - trivy fs . -f json -o trivy.json
    - trivy image $CI_REGISTRY_IMAGE -f json -o trivy-image.json
  artifacts:
    paths: [trivy.json, trivy-image.json]

semgrep-scan:
  stage: scan
  image: semgrep/semgrep
  script:
    - semgrep --config auto --json -o semgrep.json .
  artifacts:
    paths: [semgrep.json]

upload-to-vampy:
  stage: security
  needs: [trivy-scan, semgrep-scan]
  script:
    # Trivy FS
    - vampy-cli upload
        --repository $VAMPY_REPO_SLUG
        --file trivy.json
        --scanner TRIVY_FS

    # Trivy Image
    - vampy-cli upload
        --repository $VAMPY_REPO_SLUG
        --file trivy-image.json
        --scanner TRIVY_IMAGE

    # Semgrep
    - vampy-cli upload
        --repository $VAMPY_REPO_SLUG
        --file semgrep.json
        --scanner SEMGREP

    # Проверить SQG после всех загрузок
    - vampy-cli quality-gate
        --repository $VAMPY_REPO_SLUG
        --details
```

**Сценарий 3 — один сканер с кастомными параметрами + SQG:**
```yaml
semgrep-vampy:
  stage: security
  script:
    - vampy-cli scan
        --repository $VAMPY_REPO_SLUG
        --scanner SEMGREP
        --scanner-options "--config=p/owasp-top-ten"
        --check-full
        --details
```

### Quality Gate в pipeline

Vampy-cli возвращает **ненулевой exit code** если SQG не пройден — pipeline автоматически упадёт.

Пороги по умолчанию (Critical репозиторий):

| Practice | Severity | Макс. issues |
|---|---|---|
| SAST/DAST | Critical | 0 |
| SCA | Critical | 0 |
| SAST/DAST | High | 0 |
| SCA | High | 10 |

### API напрямую (без vampy-cli)

```bash
# Проверка SQG
curl -G "http://10.0.30.64/api/ext/v1/quality_gate/" \
  -H "Authentication: $VAMPY_API_TOKEN" \
  -d "repository=$VAMPY_REPO_SLUG"

# Загрузка результатов сканирования
curl -X POST "http://10.0.30.64/api/v1/scan-results/" \
  -H "Authentication: $VAMPY_API_TOKEN" \
  -F "file=@trivy.json" \
  -F "scanner=TRIVY_FS" \
  -F "repository=$VAMPY_REPO_SLUG"
```

### GitLab Webhook интеграция

1. Vampy UI → Settings → Integrations → GitLab → Add server
2. URL сервера: `https://gitlab.fgdevelop.tech`, Personal Access Token с `api` scope
3. Скопировать webhook URL: `http://10.0.30.64/api/integrations/servers/{id}/webhook/`
4. GitLab репозиторий → Settings → Webhooks → вставить URL
5. Events: **Push events**, **Merge request events**

После этого Vampy автоматически запускает сканирование при push/MR.

### Сравнение с DefectDojo-стеком

| | DefectDojo + D-Track | Vampy |
|---|---|---|
| Установка | Сложная (несколько сервисов) | Простая (docker compose) |
| RAM | 4+ GiB | ~2 GiB |
| SBOM / SCA | Dependency-Track | Trivy (встроено) |
| DAST | Сторонние инструменты | ZAP, Nuclei (встроено) |
| AI-анализ | Нет | Есть (GLM/OpenAI) |
| GitLab интеграция | Через CI | Webhook + CLI |
| Дедупликация | Да | Да (хеш-алгоритм по 4 уровням) |
| Jira | Да | Да |
| CLI для CI | Нет (curl) | vampy-cli |
| Quality Gate | SLA policies | SQG (5 preset уровней) |

---

## Официальная документация

- Trivy: https://trivy.dev/latest/docs/
- Trivy Operator: https://aquasecurity.github.io/trivy-operator/
- Dependency-Track: https://docs.dependencytrack.org/
- DefectDojo: https://defectdojo.github.io/django-DefectDojo/
- Semgrep: https://semgrep.dev/docs/
- Nova (Fairwinds): https://nova.docs.fairwinds.com/

## Источники

- https://www.reddit.com/r/kubernetes/comments/appsec_tooling
- https://owasp.org/www-project-dependency-track/
- https://github.com/DefectDojo/django-DefectDojo/discussions

## Подводные камни

- Dependency-Track требует минимум 2-4 GiB RAM (Java-приложение)
- DefectDojo: первый импорт больших JSON отчётов может занять несколько минут
- Trivy Operator: при большом кластере — высокий объём CRD; настрой `--skip-namespaces` и `--target-namespaces`
- Semgrep бесплатен для OSS, Semgrep Supply Chain (SCA) — платная фича; для SCA используй Trivy
- NVD API rate limiting: Dependency-Track использует NVD API, нужен API ключ при частых обновлениях
- DefectDojo + D-Track интеграция: D-Track → DefectDojo через "Dependency Track" connector, но он иногда лагает при > 10k компонентов
