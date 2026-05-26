# VictoriaMetrics — литьё метрик из автотестов

## 🎯 Проблема

Нужно отправлять web-vitals (LCP, FCP) и другие метрики из автотестов в существующий VictoriaMetrics-кластер, чтобы видеть их в Grafana.

## 🏗️ Куда лить

Два варианта endpoint'а на том же VMCluster:

### Prometheus text format (проще из bash)

```
POST http://prod-rbt-vm-1.rbt.ru:8480/insert/0/prometheus/api/v1/import/prometheus
```

Вход: Prometheus exposition format (обычный текстовый формат метрик).

### Remote write (для SDK / vmagent)

```
POST http://prod-rbt-vm-1.rbt.ru:8480/insert/0/prometheus/api/v1/write
```

Вход: Prometheus remote write protocol (protobuf/snappy). Использовать через SDK.

## ✅ Решение: curl одной командой

```bash
# Пуш web-vitals в VictoriaMetrics
cat <<'EOF' | curl --data-binary @- http://prod-rbt-vm-1.rbt.ru:8480/insert/0/prometheus/api/v1/import/prometheus
webvitals_lcp_ms{test="guest_checkout_cash",region="kargapole",prod="prod1",viewport="desktop",route="/basket/purchase/{id}/"} 1360
webvitals_fcp_ms{test="guest_checkout_cash",region="kargapole",prod="prod1",viewport="desktop",route="/basket/purchase/{id}/"} 564
EOF
```

## 📋 Важные лейблы

```prometheus
# source=autotests — чтобы отличать от метрик приложения
# test=guest_checkout_cash — имя сценария
# region=kargapole — регион тестирования
# prod=prod1 — целевое окружение
# viewport=desktop|mobile — тип экрана
# route=/basket/purchase/{id}/ — текущий маршрут SPA
```

## ⚠️ Что обсудить с владельцами VM

1. **Сетевой доступ** — открыт ли порт 8480 с машин автотестера до `prod-rbt-vm-1.rbt.ru`? Если нет — нужно открывать или ставить vmauth/ingress.

2. **Аутентификация** — если 8480 открыт без авторизации, это проблема: любой может писать мусор в метрики. Стоит:
   - Включить аутентификацию на vminsert
   - Или добавить `vmauth` как reverse-proxy с проверкой токена

3. **Cluster label** — vmagent добавляет `cluster="k8s-prod"` через `externalLabels`. У метрик автотестов такого лейбла не будет. Добавь `source="autotests"` чтобы фильтровать в Grafana отдельно от production-метрик.

4. **Retention** — автотестовые метрики могут быть краткосрочными (7d retention достаточен). Обсудить отдельный тенат или retention-политику.

## 🧪 Пример: функция helper для bash

```bash
# push_to_vm.sh — отправляет метрику в VictoriaMetrics
# Usage: push_to_vm test_name metric_name value [labels...]

VM_ENDPOINT="${VM_ENDPOINT:-http://prod-rbt-vm-1.rbt.ru:8480/insert/0/prometheus/api/v1/import/prometheus}"

push_metric() {
  local test="$1" metric="$2" value="$3"
  shift 3
  local labels="test=\"${test}\",source=\"autotests\""
  for l in "$@"; do
    labels="${labels},${l}"
  done
  echo "${metric}{${labels}} ${value}" | curl -s --data-binary @- "$VM_ENDPOINT"
}

# Пример использования:
push_metric "guest_checkout" "webvitals_lcp_ms" 1360 \
  "region=kargapole" "viewport=desktop" "route=/basket/purchase/{id}/"
```

## 📎 Связанные заметки

- [Архитектура VictoriaMetrics](victoria-metrics-cluster-architecture.md)
