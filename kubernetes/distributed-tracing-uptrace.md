# Distributed Tracing: от 100% error rate до первопричины за 60 секунд

**Источник:** https://habr.com/ru/articles/1008998/  
**Инструмент:** [Uptrace](https://uptrace.dev) — OpenTelemetry-native APM, open-source  
**GitHub:** https://github.com/uptrace/uptrace/  
**Demo:** http://play.uptrace.dev  

---

## Суть

OpenTelemetry + Uptrace позволяет расследовать инциденты в микросервисах за минуту по схеме:

```
Service Graph → Span Filtering → Trace Timeline → Log Correlation → Monitor
```

---

## 5 фаз расследования

### Phase 1 — Service Graph

Граф сервисов: узлы = сервисы, рёбра = вызовы между ними. Красный узел → есть ошибки.

- Два режима: **Incoming** (кто вызывает) и **Outgoing** (что вызывает сам)
- Сразу видно, какой сервис ломает цепочку

### Phase 2 — Фильтрация spans

Не смотреть на всё сразу. Последовательная фильтрация:

```
service_name = frontend      # один сервис
_kind = server               # только обработанные запросы
_status_code = error         # только ошибки
```

Затем — aggregations по группам:
```
perMin(count())   # частота проблемы
p99(_dur_ms)      # worst case latency
_error_rate       # % падений
```

### Phase 3 — Trace Timeline (Waterfall)

Полная хронология одного запроса через все сервисы.

- Ищем самый длинный span — это bottleneck
- Ищем failed spans (красные маркеры)
- Видим неожиданные вызовы

### Phase 4 — Log Correlation

Трейсы и логи связаны через `trace_id`. Вкладка **LOGS & ERRORS** в waterfall view показывает **только логи этого конкретного запроса** — без поиска по времени.

Пример первопричины из логов:
```
DNS resolution failed for product-catalog:3550
gRPC status: UNAVAILABLE (code 14)
```

Возможные причины DNS failure: сервис не запущен, неправильный hostname, NetworkPolicy блокирует DNS, DNS сервер перегружен.

### Phase 5 — Monitor

Кнопка **MONITOR** на span → автозаполненный алерт на p99 latency с нотификацией в Slack/Telegram/PagerDuty/email.

---

## Ключевые принципы

1. **Structured filtering** — не читай все логи, фильтруй последовательно
2. **Timeline thinking** — trace это история запроса, не набор логов
3. **Log-trace correlation** — `trace_id` связывает лог с полным request flow
4. **Aggregations matter** — один span не показывает паттерн; смотри p99 и error_rate
5. **От расследования → к мониторингу** — после исправления ставь алерт

---

## Uptrace

- **OpenTelemetry-native** — native support OTEL SDK без агентов
- **Self-hosted** — open-source, разворачивается в K8s или Docker
- **Хранение:** ClickHouse для spans/logs, Prometheus-совместимые метрики
- **Компоненты:** Traces, Spans, Logs & Errors, Events, Dashboards, Alerting, Service Graph

**Деплой:**
```bash
git clone https://github.com/uptrace/uptrace
cd uptrace/example/docker
docker compose up -d
# UI: http://localhost:14318
```

---

## Связь с нашими проектами

- `ya-wiki-agent` — MCP сервер уже экспортирует Prometheus метрики (`/metrics`). Uptrace можно подключить как трейсинг-бэкенд через OpenTelemetry SDK для FastAPI
- `gitlab-reviewer` / `appsec-platform` — любой микросервисный стек с OTEL инструментацией
- Альтернатива: Jaeger (только трейсы), Grafana Tempo (трейсы + Loki для логов). Uptrace — всё в одном

---

## Когда нужен distributed tracing

- Ошибка в сервисе A, причина — в сервисе C (3 уровня вглубь)
- Latency regression: непонятно где замедление
- Флапающие ошибки (не воспроизводятся локально)
- Нужна корреляция лог ↔ запрос ↔ трейс без ручного grep
