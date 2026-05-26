# VictoriaMetrics Cluster — архитектура мониторинга RBT

## 🎯 Контекст

Мониторинг-инфра в Kubernetes, VictoriaMetrics используется как внешний кластер. Сбор метрик через vmagent CRD, хранение и запросы — через внешний VM-кластер на выделенных машинах.

## 🏗️ Компоненты

| Компонент | Адрес | Назначение |
|---|---|---|
| **vminsert** | `http://prod-rbt-vm-1.rbt.ru:8480/insert/0/prometheus` | Запись метрик |
| **vmselect (primary)** | `http://prod-rbt-vm-0.rbt.ru:8481/select/0/prometheus` | Чтение (Grafana, vmalert) |
| **vmselect (replica)** | `http://prod-rbt-vm-1.rbt.ru:8481/select/0/prometheus` | Чтение (реплика для HA) |

> **0** — tenant ID в VMCluster.

```
          ┌─────────────────────────────────────────┐
          │              Kubernetes                  │
          │                                         │
          │  ┌─────────┐    ┌──────────────┐       │
          │  │ vmagent │    │ Grafana (UI) │       │
          │  └────┬────┘    └──────┬───────┘       │
          │       │               │                 │
          │  remote_write      datasource            │
          │       │               │                 │
          └───────┼───────────────┼─────────────────┘
                  │               │
                  ▼               ▼
   ┌──────────────────────────────┴──────────────┐
   │          External VMCluster                  │
   │                                              │
   │  prod-rbt-vm-1.rbt.ru:8480 (vminsert)        │
   │  prod-rbt-vm-0.rbt.ru:8481 (vmselect)        │
   │  prod-rbt-vm-1.rbt.ru:8481 (vmselect)        │
   └──────────────────────────────────────────────┘
```

## ⚙️ Как настроено в чарте

```yaml
# values.yaml чарта victoria-metrics-k8s-stack
vmsingle:
  enabled: false
vmcluster:
  enabled: false
vmagent:
  enabled: true
external:
  vm:
    write:
      url: http://prod-rbt-vm-1.rbt.ru:8480/insert/0/prometheus/api/v1/write
    read:
      url: http://prod-rbt-vm-1.rbt.ru:8481/select/0/prometheus
```

Когда `vmsingle.enabled: false` и `vmcluster.enabled: false`, чарт **автоматически** направляет vmagent remote_write на `external.vm.write.url`.

## 📥 Сбор метрик (vmagent)

vmagent собирает метрики через Kubernetes CRD:
- `VMServiceScrape` — сервисы
- `VMPodScrape` — поды
- `VMNodeScrape` — ноды

Пишет remote_write на `http://prod-rbt-vm-1.rbt.ru:8480/insert/0/prometheus/api/v1/write`.

vmagent добавляет `externalLabels`: `cluster="k8s-prod"`.

## 📖 Чтение метрик

| Клиент | Endpoint |
|---|---|
| **Grafana** | `http://prod-rbt-vm-0.rbt.ru:8481` (datasource) |
| **vmalert** | `http://prod-rbt-vm-1.rbt.ru:8481/select/0/prometheus` |

vm-0 и vm-1 — реплики vmselect для HA. Любой из них валиден для чтения.

## 📎 Связанные заметки

- [Как лить метрики из автотестов](victoria-metrics-autotest-push.md)
