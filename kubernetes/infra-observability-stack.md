# Infrastructure Observability Stack — мониторинг, алертинг, аудит, обновление образов

## Полная карта стека

```
┌─────────────────────────────────────────────────────────────────────┐
│                         ИСТОЧНИКИ ДАННЫХ                            │
│                                                                     │
│  ┌──────────┐  ┌──────────────┐  ┌─────────────────────────────┐  │
│  │   ВМ /   │  │   Docker /   │  │         Kubernetes           │  │
│  │   Хост   │  │   Compose    │  │  (API, Audit, Metrics, Logs) │  │
│  └────┬─────┘  └──────┬───────┘  └─────────────┬───────────────┘  │
└───────┼───────────────┼───────────────────────── ┼─────────────────┘
        │               │                          │
        ▼               ▼                          ▼
┌───────────────────────────────────────────────────────────────────┐
│                         СБОР ДАННЫХ                               │
│                                                                   │
│  Node Exporter    cAdvisor          kube-state-metrics            │
│  (CPU,RAM,disk)   (container CPU)   (pod/deploy state)            │
│       │               │                    │                      │
│       └───────────────┴────────────────────┘                     │
│                        Prometheus ◄── Prometheus Operator         │
│                                                                   │
│  Promtail / Fluent Bit ──────────────────────► Loki              │
│  (VM logs, Docker logs, K8s logs, Audit logs)                     │
│                                                                   │
│  Falco ──────────────────────────────────────► Falcosidekick     │
│  (K8s audit, syscalls, runtime security)              │           │
└────────────────────────────────────── ────────────────┼───────────┘
                                                         │
┌────────────────────────────────────────────────────────┼──────────┐
│                    ВИЗУАЛИЗАЦИЯ + АЛЕРТИНГ             │          │
│                                                        ▼          │
│  Grafana ◄── Prometheus ◄── AlertManager ──────► Telegram        │
│      └──── Loki                  │                    Slack       │
│      └──── Tempo (трейсы)        └──────────────► PagerDuty      │
│                                                                   │
│  Botkube ──────────────────────────────────────► Telegram        │
│  (K8s events: deploy/error/Kyverno)                               │
└───────────────────────────────────────────────────────────────────┘
        │
┌───────┼───────────────────────────────────────────────────────────┐
│       ▼         ОБНОВЛЕНИЕ ОБРАЗОВ                                │
│                                                                   │
│  Diun ──────────────────────────────────────► Telegram (notify)  │
│  (Docker Compose + K8s: оповещение о новых образах)              │
│                                                                   │
│  Watchtower ──────────────────────────────── Docker Compose      │
│  (авто-пулл + рестарт контейнеров)                               │
│                                                                   │
│  Renovate ────────────────────────────────► GitLab/GitHub MR     │
│  (Dockerfile, docker-compose.yml, Helm values, K8s manifests)    │
└───────────────────────────────────────────────────────────────────┘
```

---

## Слой 1: Метрики (Prometheus + Grafana)

### kube-prometheus-stack — всё-в-одном для K8s

Включает: Prometheus Operator, Grafana, AlertManager, Node Exporter, kube-state-metrics, cAdvisor.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  -f prometheus-values.yaml
```

```yaml
# prometheus-values.yaml
grafana:
  enabled: true
  adminPassword: "changeme"
  ingress:
    enabled: true
    hosts: [grafana.internal.company.com]

alertmanager:
  config:
    global:
      resolve_timeout: 5m
    route:
      group_by: [alertname, namespace]
      receiver: telegram
      routes:
        - match:
            severity: critical
          receiver: telegram
    receivers:
      - name: telegram
        webhook_configs:
          - url: "http://alertmanager-bot.monitoring.svc:8080"

prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          resources:
            requests:
              storage: 50Gi

nodeExporter:
  enabled: true   # метрики хоста (CPU, RAM, disk, network)

kubeStateMetrics:
  enabled: true   # состояние K8s объектов

# Добавить scrape для Docker-хостов вне K8s
additionalScrapeConfigs:
  - job_name: docker-host
    static_configs:
      - targets:
          - 10.0.30.18:9100   # Node Exporter на bare metal хосте
          - 10.0.30.18:8080   # cAdvisor
```

### Node Exporter + cAdvisor на Docker-хостах (вне K8s)

```yaml
# docker-compose.yml для monitoring агентов
services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    network_mode: host
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
      - --path.rootfs=/rootfs
      - --collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($$|/)

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    privileged: true
    devices:
      - /dev/kmsg
```

### AlertManager → Telegram

Самый простой вариант — `alertmanager-bot`:

```yaml
services:
  alertmanager-bot:
    image: metalmatze/alertmanager-bot:0.4.3
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      ALERTMANAGER_URL: http://alertmanager:9093
      TELEGRAM_ADMIN: "199636627"
      TELEGRAM_TOKEN: "<BOT_TOKEN>"
      STORE: bolt
      BOLT_PATH: /data/bot.db
    volumes:
      - alertmanager-bot-data:/data
```

---

## Слой 2: Логи (Loki + Promtail)

### Grafana Loki

```bash
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set loki.enabled=true \
  --set promtail.enabled=true \    # для K8s pod logs
  --set grafana.enabled=false      # уже установлена
```

### Promtail на Docker-хостах (вне K8s)

```yaml
# /etc/promtail/config.yml
server:
  http_listen_port: 9080

clients:
  - url: http://loki.monitoring.svc:3100/loki/api/v1/push
    # или внешний адрес если хост вне K8s:
    # url: http://10.0.30.18:3100/loki/api/v1/push

scrape_configs:
  # Docker контейнеры
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: [__meta_docker_container_name]
        target_label: container
      - source_labels: [__meta_docker_compose_project]
        target_label: compose_project
      - source_labels: [__meta_docker_compose_service]
        target_label: compose_service

  # Системные логи
  - job_name: syslog
    static_configs:
      - targets: [localhost]
        labels:
          job: syslog
          host: minisforum
          __path__: /var/log/syslog

  # K8s Audit logs
  - job_name: k8s-audit
    static_configs:
      - targets: [localhost]
        labels:
          job: k8s-audit
          __path__: /var/log/kubernetes/audit/*.log
    pipeline_stages:
      - json:
          expressions:
            user: user.username
            verb: verb
            resource: objectRef.resource
            namespace: objectRef.namespace
      - labels:
          user:
          verb:
          resource:
          namespace:
```

---

## Слой 3: Kubernetes Audit + Security (Falco)

> Подробно: `kubernetes/kubernetes-audit-logging.md`

```bash
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.telegram.token="<TOKEN>" \
  --set falcosidekick.config.telegram.chatid="<CHAT_ID>"
```

---

## Слой 4: K8s Events (Botkube)

> Подробно: `kubernetes/botkube-policy-reporter.md`

Алерты о деплоях, ошибках подов, Kyverno PolicyReport.

---

## Слой 5: Обновление образов

### Diun — уведомления о новых образах (Docker Compose + K8s)

**Только оповещает**, не изменяет ничего. Поддерживает Docker socket, Docker Compose, Kubernetes.

```yaml
# docker-compose.yml
services:
  diun:
    image: crazymax/diun:latest
    container_name: diun
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - diun-data:/data
      - ./diun.yml:/diun.yml:ro
    environment:
      TZ: Europe/Saratov
      LOG_LEVEL: info
      LOG_JSON: "false"
```

```yaml
# diun.yml
watch:
  schedule: "0 8 * * *"    # проверять каждый день в 8:00
  jitter: 30s
  firstCheckNotif: false

notif:
  telegram:
    token: "<BOT_TOKEN>"
    chatIDs:
      - "199636627"
    templateBody: |
      🐳 Новый образ: {{ .Entry.Image }}
      Тег: {{ .Entry.Manifest.Tag }}
      Дата: {{ .Entry.Manifest.Created }}

providers:
  docker:
    watchStopped: true    # следить даже за остановленными контейнерами

  # Kubernetes (если запущен в K8s)
  kubernetes:
    watchAll: true
```

```yaml
# Разметить контейнеры для слежки в docker-compose.yml
services:
  myapp:
    image: myapp:latest
    labels:
      - "diun.enable=true"
      - "diun.regopt=myreg"     # опционально: настройки registry
```

---

### Watchtower — авто-обновление Docker Compose

**Автоматически** пуллит новые образы и рестартует контейнеры.

```yaml
services:
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      TZ: Europe/Saratov
      WATCHTOWER_SCHEDULE: "0 0 4 * * *"     # в 4 утра
      WATCHTOWER_CLEANUP: "true"              # удалять старые образы
      WATCHTOWER_INCLUDE_STOPPED: "false"
      WATCHTOWER_NOTIFICATIONS: shoutrrr
      WATCHTOWER_NOTIFICATION_URL: "telegram://<BOT_TOKEN>@telegram?channels=<CHAT_ID>"
      # Режим только уведомлений (не обновлять автоматически):
      # WATCHTOWER_MONITOR_ONLY: "true"
```

**Выборочное обновление** — по умолчанию обновляет ВСЕ контейнеры.
Чтобы обновлять только отмеченные:

```yaml
environment:
  WATCHTOWER_LABEL_ENABLE: "true"   # обновлять только с меткой

services:
  nginx:
    image: nginx:latest
    labels:
      - "com.centurylinklabs.watchtower.enable=true"   # следить
  db:
    image: postgres:15
    labels:
      - "com.centurylinklabs.watchtower.enable=false"  # не трогать
```

---

### Renovate — обновление через MR/PR (GitOps-подход)

Создаёт MR в GitLab / PR в GitHub с обновлёнными версиями образов.
Работает с: `Dockerfile`, `docker-compose.yml`, Helm `values.yaml`, K8s manifests.

```bash
# Self-hosted через GitLab CI
# .gitlab-ci.yml
renovate:
  image: renovate/renovate:latest
  script:
    - renovate
  variables:
    RENOVATE_TOKEN: $GITLAB_TOKEN
    RENOVATE_GIT_AUTHOR: "Renovate Bot <renovate@company.com>"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
```

```json
// renovate.json в корне репо
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "assignees": ["your-username"],
  "reviewers": ["your-username"],
  "prConcurrentLimit": 5,
  "packageRules": [
    {
      "matchDatasources": ["docker"],
      "automerge": false,        // не мержить автоматически — только создавать MR
      "labels": ["dependencies", "docker"]
    },
    {
      "matchPackageNames": ["postgres", "redis"],
      "automerge": false,
      "schedule": ["before 6am on monday"]   // обновлять только по понедельникам
    }
  ]
}
```

---

### Keel — авто-обновление образов в Kubernetes

Обновляет `image:tag` в Deployments автоматически при появлении нового образа.

```bash
helm repo add keel https://charts.keel.sh
helm install keel keel/keel --namespace keel --create-namespace
```

```yaml
# Аннотация на Deployment
metadata:
  annotations:
    keel.sh/policy: minor           # major | minor | patch | all | force
    keel.sh/trigger: poll           # poll registry или push webhook
    keel.sh/pollSchedule: "@every 24h"
    keel.sh/notify: "telegram"      # уведомить при обновлении
```

---

## Итоговая таблица

| Задача | Инструмент | Где |
|--------|-----------|-----|
| Метрики ВМ/хоста | Node Exporter → Prometheus | все хосты |
| Метрики Docker/cAdvisor | cAdvisor → Prometheus | Docker-хосты |
| Метрики K8s | kube-state-metrics → Prometheus | K8s |
| Все метрики — дашборды | Grafana | centralised |
| Алерты по метрикам | AlertManager → Telegram | centralised |
| Логи всего | Loki + Promtail/Fluent Bit | centralised |
| K8s audit real-time | Falco → Falcosidekick → Telegram | K8s |
| K8s события (деплои) | Botkube → Telegram | K8s |
| Kyverno политики | Policy Reporter | K8s |
| Docker образы — уведомления | Diun → Telegram | Docker-хосты |
| Docker образы — авто-update | Watchtower | Docker-хосты |
| K8s образы — MR | Renovate | GitLab/GitHub |
| K8s образы — авто-update | Keel | K8s |

---

## Минимальный старт (приоритеты)

**Шаг 1 — Метрики и алерты (самое важное):**
```
kube-prometheus-stack + Node Exporter на хостах + AlertManager → Telegram
```

**Шаг 2 — Логи:**
```
Loki + Promtail (на хостах + в K8s)
```

**Шаг 3 — Уведомления об обновлениях образов:**
```
Diun (уведомления) + Watchtower в monitor-only (без авто-апдейта)
```

**Шаг 4 — K8s audit и security:**
```
Falco + Falcosidekick → Telegram
```

**Шаг 5 — K8s события:**
```
Botkube → Telegram
```

---

## Официальные источники

- kube-prometheus-stack: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
- Loki: https://grafana.com/docs/loki/
- Falco: https://falco.org
- Botkube: https://docs.botkube.io
- Diun: https://crazymax.dev/diun/
- Watchtower: https://containrrr.dev/watchtower/
- Renovate: https://docs.renovatebot.com
- Keel: https://keel.sh/docs/
