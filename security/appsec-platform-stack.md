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
