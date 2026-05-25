# Prometheus Pushgateway

## 🎯 Проблема

Есть метрики из CI-джоб, автотестов, cron-задач — процессов с коротким жизненным циклом.
Prometheus не может их заскрейпить по pull-модели: пока скрейпер просыпается, джоба уже завершилась.
Нужен способ «втолкнуть» метрики из эфемерных процессов в Prometheus.

## ✅ Решение

Pushgateway — sidecar-сервис, который принимает push-запросы с метриками и отдаёт их Prometheus через endpoint `/metrics`.

### Как это работает

```
CI Job → push → Pushgateway → scrape → Prometheus → Grafana
```

1. Джоба завершается и пушит метрики в Pushgateway через HTTP API
2. Prometheus скрейпит Pushgateway как обычный target
3. Метрики доступны в Grafana как обычные time-series

### Docker Compose

```yaml
# pushgateway/docker-compose.yml
version: '3.8'
services:
  pushgateway:
    image: prom/pushgateway:latest
    container_name: pushgateway
    command:
      - '--web.listen-address=:9091'
      - '--persistence.file=/data/metrics.dat'
      - '--persistence.interval=5m'
    ports:
      - '9091:9091'
    volumes:
      - pushgateway_data:/data
    restart: unless-stopped

volumes:
  pushgateway_data:
```

### Prometheus scrape config

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'pushgateway'
    honor_labels: true      # сохраняет labels из пуша
    scrape_interval: 60s
    static_configs:
      - targets: ['pushgateway:9091']
```

`honor_labels: true` критично — без него Prometheus добавит свои `instance`/`job` labels поверх твоих, сломав группировку.

### API — пуш метрик

```bash
# Базовый пуш (POST — заменяет только переданные метрики)
cat <<EOF | curl --data-binary @- http://localhost:9091/metrics/job/ci_web_vitals/instance/build_abc123
# HELP web_vital_lcp_ms LCP in ms
# TYPE web_vital_lcp_ms gauge
web_vital_lcp_ms{branch="main",region="msk"} 2340
EOF

# PUT — заменяет ВСЕ метрики для этой job+instance
curl -X PUT --data-binary @- http://localhost:9091/metrics/job/ci_web_vitals/instance/build_abc123 < metrics.txt

# DELETE — удалить instance (чтобы не скапливался мусор)
curl -X DELETE http://localhost:9091/metrics/job/ci_web_vitals/instance/build_abc123
```

### Пример: Web Vitals из CI в Pushgateway

```bash
#!/bin/bash
# push_perf_metrics.sh
JOB="web_vitals"
INSTANCE="${BUILD_ID}_${REGION}_${VIEWPORT}"
PUSHGATEWAY="http://pushgateway:9091"

# Чистим предыдущий пуш для этого instance
curl -s -X DELETE "${PUSHGATEWAY}/metrics/job/${JOB}/instance/${INSTANCE}" >/dev/null 2>&1

# Пушим
curl -s --data-binary @- "${PUSHGATEWAY}/metrics/job/${JOB}/instance/${INSTANCE}" <<EOF
# HELP web_vital_lcp_ms LCP in ms
# TYPE web_vital_lcp_ms gauge
web_vital_lcp_ms{region="${REGION}",viewport="${VIEWPORT}",prod="${PROD}",git_sha="${GIT_SHA}",test="${TEST_NAME}",status="${S_LCP}"} ${LCP}
# HELP web_vital_cls Cumulative Layout Shift
# TYPE web_vital_cls gauge
web_vital_cls{region="${REGION}",viewport="${VIEWPORT}",prod="${PROD}",git_sha="${GIT_SHA}",test="${TEST_NAME}",status="${S_CLS}"} ${CLS}
# HELP web_vital_fcp_ms First Contentful Paint
# TYPE web_vital_fcp_ms gauge
web_vital_fcp_ms{region="${REGION}",viewport="${VIEWPORT}",prod="${PROD}",git_sha="${GIT_SHA}",test="${TEST_NAME}",status="${S_FCP}"} ${FCP}
# HELP web_vital_ttfb_ms Time to First Byte
# TYPE web_vital_ttfb_ms gauge
web_vital_ttfb_ms{region="${REGION}",viewport="${VIEWPORT}",prod="${PROD}",git_sha="${GIT_SHA}",test="${TEST_NAME}",status="${S_TTFB}"} ${TTFB}
# HELP network_dns_ms DNS lookup
# TYPE network_dns_ms gauge
network_dns_ms{region="${REGION}",viewport="${VIEWPORT}",prod="${PROD}",git_sha="${GIT_SHA}",test="${TEST_NAME}"} ${DNS}
# HELP network_tcp_ms TCP connect
# TYPE network_tcp_ms gauge
network_tcp_ms{region="${REGION}",viewport="${VIEWPORT}",prod="${PROD}",git_sha="${GIT_SHA}",test="${TEST_NAME}"} ${TCP}
# HELP network_ssl_ms SSL handshake
# TYPE network_ssl_ms gauge
network_ssl_ms{region="${REGION}",viewport="${VIEWPORT}",prod="${PROD}",git_sha="${GIT_SHA}",test="${TEST_NAME}"} ${SSL}
EOF
```

### Kubernetes

```yaml
# pushgateway/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pushgateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pushgateway
  template:
    metadata:
      labels:
        app: pushgateway
    spec:
      containers:
        - name: pushgateway
          image: prom/pushgateway:latest
          ports:
            - containerPort: 9091
          args:
            - '--persistence.file=/data/metrics.dat'
            - '--persistence.interval=5m'
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: pushgateway-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: pushgateway
  labels:
    app: pushgateway
spec:
  ports:
    - port: 9091
      targetPort: 9091
  selector:
    app: pushgateway

# Prometheus ServiceMonitor (если prometheus-operator)
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: pushgateway
spec:
  selector:
    matchLabels:
      app: pushgateway
  endpoints:
    - port: '9091'
      honorLabels: true
```

## 📎 Официальная документация

- [Prometheus Pushgateway](https://github.com/prometheus/pushgateway)
- [When to use Pushgateway](https://prometheus.io/docs/practices/pushing/)
- [Instrumenting: Pushing metrics](https://prometheus.io/docs/instrumenting/pushing/)

## 🔗 Источники

- [Prometheus Pushgateway module — Deckhouse](https://deckhouse.io/modules/prometheus-pushgateway/examples.html)
- [Docker Compose Template (ifengx.com)](https://docker.weifengx.com/app/prometheus-pushgateway)

## 💡 Best Practices

**Do:**
- Используй `honor_labels: true` в scrape config
- Всегда задавай уникальные `instance` labels (git_sha, build_id)
- Чисти старые instance через DELETE API перед новым пушем
- Пуши только **Gauge** метрики (counter/histogram из эфемерных процессов — бессмысленно)
- Для batch jobs без instance-привязки — не включай machine/instance label вообще

**Don't:**
- Не используй Pushgateway для постоянно работающих сервисов (им нужен нормальный scrape)
- Не заменяй pull-модель Prometheus пушингом ради «удобства»
- Не гоняй Pushgateway как HA-сервис (single instance — ОК, это не критичный компонент)
- Не пуши тысячи уникальных time-series за раз — Pushgateway не масштабируется горизонтально
- Не храни метрики в Pushgateway вечно — данные живут пока их не удалят, нужен lifecycle management

## ⚠️ Подводные камни

- **Метрики не удаляются автоматически.** Pushgateway помнит всё, что в него запушили, пока не удалишь явно. Если CI-джоба пушила с instance=`build_42` и этот build давно не актуален — метрика продолжит висеть. Решение: `-X DELETE` перед каждым пушем.
- **Single point of failure.** Если Pushgateway упадёт, метрики из текущего прогона потеряются. Но для CI это допустимо — следующий билд запушит новые.
- **Нет `up` метрики.** Prometheus генерирует `up` только для pull-целей. При пушинг-модели ты теряешь health-check. Добавляй собственную heartbeat-метрику если нужно.
- **Расход памяти.** Pushgateway держит все метрики в памяти (+ опционально persistence на диск). Миллионы уникальных instance сожрут RAM.
- **Не для стриминга.** Частые пуши (каждые 5 секунд) — плохая идея. Pushgateway рассчитан на эпизодические пуши от завершённых джоб.
- **Kubernetes — 1 реплика.** HPA не имеет смысла. Pushgateway stateful по природе, не шардируется.
- **Метрики-сироты.** Если instance перестал пушить и ты не удалил старые метрики — они будут в Grafana вечно. Нужен отдельный cleanup job/скрипт.

## 🔗 Связанные заметки

- [Infra Observability Stack (Prometheus + Grafana + VictoriaMetrics)](/kubernetes/infra-observability-stack.md) — полный мониторинг-стек
- [Logging with Victorialogs Stack](/kubernetes/logging-victorialogs-stack.md) — логирование вместо Loki
- [Secrets management в CI/CD](/security/secrets-management-cicd-containers.md) — как не слить токены при пуше метрик

---

## 🚀 Альтернатива: VictoriaMetrics без Pushgateway

Если в стеке уже есть **VictoriaMetrics** (или VictoriaMetrics Cluster), Pushgateway не нужен. VM умеет принимать метрики напрямую через несколько протоколов push. Это избавляет от лишнего компонента и lifecycle-головной боли.

### Механика

```
CI Job → push напрямую в /write → VictoriaMetrics → Grafana
```

VM принимает метрики push-методом через:
- **InfluxDB line protocol** (`/write`)
- **Prometheus remote write** (`/api/v1/write`)
- **Graphite plaintext** (`/graphite`)
- **JSON import** (`/api/v1/import`)

Pushgateway **не нужен**. VM сам управляет retention (`--retentionPeriod`), метрики не зависают вечно.

### Пример: Web Vitals JSON → VictoriaMetrics (InfluxDB line protocol)

Исходный JSON (из автотестов):

```json
{
 "test_name": "test_guest_checkout_from_item_page_cash[prod1:kargapole-desktop]",
 "url": "https://kargapole.rbt.ru/basket/purchase/...",
 "viewport": "desktop",
 "region": "kargapole",
 "prod": "prod1",
 "git_sha": "3f0567b",
 "lcp": 1360,
 "cls": 0.011586077982420825,
 "inp": 192,
 "fcp": 564,
 "ttfb": 366.7,
 "dns": 6.2,
 "tcp": 127.5,
 "ssl": 102.6,
 "redirect": 0,
 "dom_content_loaded": 1943.1,
 "load_event": 2117.6,
 "hydration": 752.6,
 "extras": { "navigations_count": 5 },
 "status": {
   "lcp": "good", "cls": "good", "inp": "good",
   "fcp": "good", "ttfb": "good", "dns": "good",
   "tcp": "ni", "ssl": "good", "redirect": "good",
   "dom_content_loaded": "ni", "load_event": "good", "hydration": "good"
 }
}
```

Скрипт в CI-пайплайне (одна команда curl, без Pushgateway):

```bash
#!/bin/bash
# push_to_vm.sh — пуш метрик из JSON напрямую в VictoriaMetrics

JSON_FILE="$1"
VM_URL="${VM_URL:-http://victoriametrics:8428}"

# Извлекаем значения через jq
TEST=$(jq -r '.test_name' "$JSON_FILE" | tr ' ' '_')
REGION=$(jq -r '.region' "$JSON_FILE")
VIEWPORT=$(jq -r '.viewport' "$JSON_FILE")
PROD=$(jq -r '.prod' "$JSON_FILE")
GIT_SHA=$(jq -r '.git_sha' "$JSON_FILE")

LABELS="region=${REGION},viewport=${VIEWPORT},prod=${PROD},git_sha=${GIT_SHA},test=${TEST}"

# InfluxDB line protocol: measurement labels key=val,key=val timestamp
curl -s -X POST "${VM_URL}/write" --data-binary "
web_vitals,${LABELS} lcp=$(jq '.lcp' "$JSON_FILE"),cls=$(jq '.cls' "$JSON_FILE"),fcp=$(jq '.fcp' "$JSON_FILE"),ttfb=$(jq '.ttfb' "$JSON_FILE"),inp=$(jq '.inp' "$JSON_FILE")
network,${LABELS} dns=$(jq '.dns' "$JSON_FILE"),tcp=$(jq '.tcp' "$JSON_FILE"),ssl=$(jq '.ssl' "$JSON_FILE"),redirect=$(jq '.redirect' "$JSON_FILE")
browser,${LABELS} dom_content_loaded=$(jq '.dom_content_loaded' "$JSON_FILE"),load_event=$(jq '.load_event' "$JSON_FILE"),hydration=$(jq '.hydration' "$JSON_FILE")
test_meta,${LABELS} navigations_count=$(jq '.extras.navigations_count // 0' "$JSON_FILE")
"

echo "Pushed to ${VM_URL}/write"
```

### PromQL-запросы в Grafana

Те же метрики, что и через Pushgateway, но с другим source:

```promql
# LCP по регионам за последний час
avg(web_vitals_lcp{region=~"$region"}) by (region, test)

# Trend: LCP за неделю
avg_over_time(web_vitals_lcp{test=~"$test"}[7d])

# Только «плохие» прогоны (статус был в Pushgateway-версии, здесь через threshold)
web_vitals_lcp{viewport="desktop"} > 2500
```

### Pushgateway vs VictoriaMetrics direct push

| Критерий | Pushgateway | VictoriaMetrics /write |
|---|---|---|
| Посредник | ✅ нужен | ❌ не нужен |
| Lifecycle метрик | Надо чистить руками | Автоматический retention |
| Память | Все метрики в RAM | Эффективное хранение на диске |
| Долгосрочное хранение | Нет | Да (retentionPeriod) |
| Масштабирование | 1 реплика (SPOF) | VM Cluster — горизонтально |
| Метрики-сироты | Вечная проблема | Удаляются по TTL |
| Протоколы | Только Prometheus exposition | InfluxDB, Prometheus remote write, Graphite, JSON |
| Даунсэмплинг | Нет | Да (1m→5m→1h) |

### Как выбрать

**Pushgateway** — если:
- Стек строго Prometheus, VM не планируется
- Нужны метрики в PromQL из коробки без смены формата
- Простота «закинул и забыл» на маленьком масштабе

**VictoriaMetrics /write** — если:
- VM уже есть в стеке
- Важна долгосрочная история
- Не хочется решать проблему «как чистить старые instance»
- Нужен даунсэмплинг и эффективное хранение

## 📎 Дополнительные источники

- [VictoriaMetrics — How to import data](https://docs.victoriametrics.com/#how-to-import-time-series-data)
- [InfluxDB line protocol spec](https://docs.influxdata.com/influxdb/v2/reference/syntax/line-protocol/)
