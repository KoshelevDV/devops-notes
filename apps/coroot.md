# Coroot — eBPF Observability Platform

**Сайт:** https://coroot.com  
**GitHub:** https://github.com/coroot/coroot  
**Лицензия:** Apache 2.0 (Community Edition)  
**Дата заметки:** 2026-03-28

---

## Что это

Open source observability платформа на eBPF. Агент перехватывает трафик на уровне ядра Linux — **без изменений в коде приложений**, без service mesh, без OpenTelemetry instrumentation.

**Минимальный Linux kernel:** 5.1+

---

## Что умеет (Community Edition)

| Возможность | Описание |
|---|---|
| **Service Map** | Автоматическая карта сервисов с метриками на связях (RPS, latency, errors) |
| **Протоколы** | HTTP, gRPC, PostgreSQL, MySQL, Redis, Memcached, MongoDB, Kafka, RabbitMQ — парсятся автоматически |
| **Slow queries** | SQL запросы к БД видны прямо в карте, без изменений в приложении |
| **Метрики** | CPU, RAM, диск, сеть по каждому контейнеру/процессу |
| **Логи** | Aggregation + pattern clustering, journald, docker, containerd |
| **Distributed tracing** | Через eBPF без instrumentation; OpenTelemetry тоже поддерживается |
| **Continuous profiling** | CPU профили приложений в 1 клик |
| **SLO tracking** | Автоматические предложения SLO по реальному трафику |
| **Deployment tracking** | Автоматически сравнивает релизы между собой |
| **Health inspections** | 80+ автоматических проверок, единый алерт с контекстом |

**Ограничения CE (нет в бесплатной):**
- SSO (SAML/OIDC)
- RBAC
- AI root cause analysis

---

## Архитектура

```
[VM / Baremetal]          [K8s nodes]
coroot-node-agent         coroot-node-agent (DaemonSet)
    │ eBPF                     │ eBPF
    │ metrics/traces            │ metrics/traces
    └──────────┬────────────────┘
               ▼
        [Coroot Server]
        [ClickHouse]          ← хранилище метрик, трейсов, логов
               │
         [Web UI :8080]
```

**Один сервер — все окружения.** Разделение через Projects.

---

## Разделение окружений

**По Projects (рекомендуется):**
- Production → Project "prod"
- Staging → Project "staging"
- Агенты каждого окружения шлют данные в разные project_id

**Маркировка VM-агентов через env:**
```bash
-e COROOTENV_environment=production
-e COROOTENV_datacenter=dc1
-e COROOTENV_team=backend
```
Теги появляются в UI, фильтруются в карте.

**K8s — автоматически:**
- Namespace используется как группа (`ns:prod`, `ns:monitoring`)
- Labels пода передаются в метрики

---

## Установка

### Сервер (Docker)

```bash
docker run -d --name coroot \
  --restart=unless-stopped \
  -p 8080:8080 \
  -v /opt/coroot-data:/data \
  ghcr.io/coroot/coroot \
  --listen=:8080 \
  --data-dir=/data
```

### Агент на VM (Docker, минимальный)

```bash
docker run -d --name coroot-node-agent \
  --restart=unless-stopped \
  --privileged \
  --pid=host \
  --network=host \
  -v /sys/kernel/tracing:/sys/kernel/tracing \
  -v /sys/kernel/debug:/sys/kernel/debug \
  -v /sys/fs/cgroup:/sys/fs/cgroup \
  ghcr.io/coroot/coroot-node-agent \
  --collector-endpoint=http://<COROOT_SERVER>:8080
```

### Агент на VM (Ansible role)

```yaml
# roles/coroot-node-agent (в infra-monitoring)
- hosts: all
  roles:
    - coroot-node-agent
  vars:
    coroot_server_url: "http://10.0.30.18:8080"
    coroot_environment: "production"
```

### K8s (Helm)

```bash
helm repo add coroot https://coroot.github.io/helm-charts
helm repo update

# Сервер + агент (DaemonSet) + Prometheus + ClickHouse
helm install coroot coroot/coroot \
  --namespace coroot \
  --create-namespace \
  -f values-coroot.yaml
```

---

## Helm values (ключевые)

```yaml
# values-coroot.yaml
corootCE:
  enabled: true
  replicas: 1
  resources:
    requests:
      cpu: 200m
      memory: 1Gi
  persistentVolume:
    size: 20Gi
    storageClassName: "local-path"
  ingress:
    enabled: true
    className: nginx
    hostname: coroot.example.com

corootClusterAgent:
  enabled: true
  config:
    coroot_url: http://coroot:8080

# Встроенный Prometheus (можно использовать свой)
prometheus:
  enabled: true
  server:
    retention: 7d
    persistentVolume:
      size: 20Gi

# ClickHouse (для логов и трейсов)
clickhouse:
  enabled: true
  persistence:
    size: 50Gi
```

---

## Ansible role — coroot-node-agent

Роль находится в `infra-monitoring/ansible/roles/coroot-node-agent/`

```bash
# Запуск
ansible-playbook ansible/playbooks/coroot.yml -i inventories/production/
```

---

## Требования к хостам для агента

| Параметр | Значение |
|---|---|
| Linux kernel | 5.1+ (рекомендуется 5.15+) |
| Привилегии | `--privileged` + `CAP_BPF`, `CAP_SYS_ADMIN` |
| Docker / containerd | для мониторинга контейнеров |
| Доступ к `/sys/kernel/tracing` | обязателен |

---

## Потребление ресурсов

| Компонент | CPU | RAM | Диск |
|---|---|---|---|
| Coroot сервер | ~200m | ~1 ГБ | 10-50 ГБ |
| ClickHouse | ~500m | ~2 ГБ | ~1 ГБ/хост/месяц |
| Агент на ноде | ~50-100m | ~100 МБ | — |

---

## Ссылки

- Официальный Helm chart: https://github.com/coroot/helm-charts
- Агент GitHub: https://github.com/coroot/coroot-node-agent
- Документация: https://docs.coroot.com/
- Live demo: https://demo.coroot.com/
- Блог (eBPF service map): https://coroot.com/blog/building-a-service-map-using-ebpf
