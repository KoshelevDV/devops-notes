# Botkube + Policy Reporter — алерты о деплоях и Kyverno-политиках

## Проблема

Нужно получать уведомления о:
- Деплоях (что задеплоено, в каком namespace, кем)
- Срабатывании/нарушении Kyverno-политик
- Ошибках подов (CrashLoopBackOff, OOMKilled)

## Botkube

Kubernetes-нотификатор с поддержкой Telegram, Slack, Teams, Discord, webhook.
Следит за Kubernetes Events и изменениями ресурсов.

### Установка

```bash
helm repo add botkube https://charts.botkube.io
helm repo update

helm install botkube botkube/botkube \
  --namespace botkube --create-namespace \
  -f botkube-values.yaml
```

### values.yaml

```yaml
communications:
  default-group:
    telegram:
      enabled: true
      token: "<BOT_TOKEN>"        # токен от @BotFather
      chatID: "<CHAT_ID>"         # ID чата / канала

sources:
  k8s-all-events:
    botkube/kubernetes:
      config:
        resources:
          # Деплои
          - type: Deployment
            namespaces:
              include: [".*"]
            events:
              - create
              - update
              - delete
              - error
          # StatefulSet / DaemonSet
          - type: StatefulSet
            events: [create, update, delete, error]
          - type: DaemonSet
            events: [create, update, delete, error]
          # Проблемные поды
          - type: Pod
            events:
              - error          # CrashLoopBackOff, OOMKilled, ImagePullBackOff
          # Kyverno PolicyReport
          - type: PolicyReport
            namespaces:
              include: [".*"]
            events:
              - create
              - update
          - type: ClusterPolicyReport
            events:
              - create
              - update

executors:
  k8s-default-tools:
    botkube/kubectl:
      enabled: true              # разрешить kubectl-команды из чата
      config:
        defaultNamespace: default
        restrictAccess: true     # только read-only команды

# Фильтрация: не спамить в рабочее время
filters:
  kubernetes:
    objectAnnotationChecker: true
    nodeEventsChecker: false
```

### Пример уведомления в Telegram

```
🚀 Deployment nginx-app created in namespace production
   Image: registry.company.com/nginx:1.25.3
   Replicas: 3
   By: user@company.com
```

### Kubectl из Telegram

```
# В чате с ботом:
@botkube kubectl get pods -n production
@botkube kubectl describe deployment nginx-app -n production
```

---

## Policy Reporter

Специализированный инструмент для отображения и отправки Kyverno PolicyReport.
Имеет веб-UI и поддержку различных каналов уведомлений.

### Установка

```bash
helm repo add policy-reporter https://kyverno.github.io/policy-reporter
helm repo update

helm install policy-reporter policy-reporter/policy-reporter \
  --namespace policy-reporter --create-namespace \
  -f policy-reporter-values.yaml
```

### values.yaml

```yaml
ui:
  enabled: true
  # kubectl port-forward svc/policy-reporter-ui 8082:8080 -n policy-reporter

kyvernoPlugin:
  enabled: true

# Prometheus метрики
monitoring:
  enabled: true   # ServiceMonitor для Prometheus Operator

target:
  # Slack
  slack:
    webhook: "https://hooks.slack.com/services/..."
    minimumSeverity: "medium"   # low | medium | high | critical
    skipExistingOnStartup: true
    channels:
      - channel: "#k8s-security"
        minimumSeverity: "high"

  # Произвольный webhook (AlertManager, свой сервис, n8n и т.п.)
  webhook:
    host: "https://alertmanager.internal.company.com/api/v2/alerts"
    minimumSeverity: "medium"
    skipExistingOnStartup: true
    headers:
      Authorization: "Bearer <token>"

  # Loki (для хранения в Grafana Loki)
  loki:
    host: "http://loki.monitoring.svc:3100"
    minimumSeverity: "warning"
    skipExistingOnStartup: true
    customLabels:
      cluster: production
```

### Prometheus AlertManager — алерт на нарушения Kyverno

```yaml
# prometheus alert rule
groups:
  - name: kyverno
    rules:
      - alert: KyvernoPolicyViolation
        expr: |
          increase(policy_report_result{result="fail"}[5m]) > 0
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "Kyverno: нарушена политика {{ $labels.policy }}"
          description: "Rule: {{ $labels.rule }}, Namespace: {{ $labels.namespace }}"

      - alert: KyvernoHighSeverityViolation
        expr: |
          increase(policy_report_result{result="fail", severity="high"}[5m]) > 0
        labels:
          severity: critical
```

---

## Схема взаимодействия

```
Kubernetes API
      │
      ├─► Kyverno Webhook ──► PolicyReport CRD
      │                              │
      │                    Policy Reporter ──► Slack / Webhook / Loki
      │
      └─► Kubernetes Events
                 │
              Botkube ──► Telegram / Slack / Teams
```

---

## Официальные источники

- Botkube docs: https://docs.botkube.io
- Botkube GitHub: https://github.com/kubeshop/botkube
- Policy Reporter docs: https://kyverno.github.io/policy-reporter/
- Policy Reporter GitHub: https://github.com/kyverno/policy-reporter

---

## Подводные камни

1. **Botkube спамит** — по умолчанию присылает ВСЕ события. Обязательно настроить фильтрацию по namespace и типу событий.
2. **chatID для Telegram** — нужен числовой ID, не username. Получить: написать боту, зайти на `https://api.telegram.org/bot<TOKEN>/getUpdates`.
3. **Policy Reporter и `skipExistingOnStartup`** — без этого при старте пришлёт уведомления по всем существующим PolicyReport (может быть шторм).
4. **Botkube RBAC** — по умолчанию kubectl из чата выполняется с правами ClusterAdmin. Ограничить через `restrictAccess: true` и отдельный ServiceAccount.
