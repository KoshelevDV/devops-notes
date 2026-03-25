# Логирование в Kubernetes: VictoriaLogs стек

> Опыт реальной эксплуатации и бенчмарки log collectors. Путь: Loki → ClickHouse → VictoriaLogs.

---

## Эволюция стека (опыт Lenta tech, >2000 нод, >1 TB логов/сутки)

### Этап 1: Promtail + Loki + Grafana
- Развернули за день, полгода работало нормально
- **Проблемы при росте:** нелинейный рост потребления ресурсов, таймауты сложных запросов, нет отказоустойчивости

### Этап 2: Vector + Kafka + ClickHouse
- Vector — агент сбора логов (заменил Promtail)
- Kafka — буферный слой (хранит несколько дней на случай падений)
- ClickHouse — финальное хранилище (аналитика, скорость)
- **Начальный результат:** потребление агентов упало в разы, 11 vCPU / 16 GB справлялись

**Через 2 года появились проблемы:**
- Кластер ClickHouse вырос до 3 инстансов по 16 vCPU / 48 GB каждый
- MergeTree при пиках не успевал — росли куски, падала скорость чтения
- ZooKeeper терял данные при дедупликации → неконсистентность реплик
- Сотни микросервисов = постоянно новые форматы логов = усложнение схемы

> **Вывод:** в экосистеме из сотен микросервисов жёсткая стандартизация формата логов бессмысленна. Нужна система, способная "переваривать" любой хаос входящих данных.

### Этап 3: vlagent + VictoriaLogs
- **Сжатие данных: 50x** по сравнению с ClickHouse
- Стабильная работа под нагрузкой
- Переход в production занял дни, не недели
- Поддерживает любые форматы логов без схемы

---

## Бенчмарк log collectors (VictoriaMetrics, 2026)

**Среда:** GCloud n2-highcpu-32 (32 vCPU, 32 GiB RAM), single-node kind k8s, 100 Pods

**Методология:** JSON логи с `sequence_id` → детектирование потерь по пропускам в последовательности

### Результаты throughput (100 Pods)

| Collector | Max logs/sec | vs лидер |
|---|---|---|
| **vlagent** | **143 000** | 1.0x |
| Fluent Bit | 31 300 | 4.5x |
| Vector | 25 000 | 5.7x |
| OTel Collector | 20 500 | 6.9x |
| Alloy | 15 700 | 9.1x |
| Grafana Agent | 14 800 | 9.7x |
| Promtail | 13 400 | 10.6x |
| Filebeat | 5 250 | 27.2x |
| Fluentd | 5 100 | 28.0x |

### CPU при 10k logs/sec

| Collector | CPU cores | vs лидер |
|---|---|---|
| **vlagent** | **0.062** | 1.0x |
| Fluent Bit | 0.260 | 4.2x |
| Vector | 0.412 | 6.6x |
| OTel Collector | 0.491 | 7.9x |
| Grafana Agent | 0.552 | 8.9x |

**Условия:** одинаковые лимиты для всех (1 CPU core, 1 GiB RAM), дефолтные конфиги Helm chart, без тюнинга.

Исходники бенчмарка: https://github.com/VictoriaMetrics/log-collectors-benchmark

---

## Когда что выбирать

| Ситуация | Рекомендация |
|---|---|
| Уже используешь VictoriaMetrics | vlagent + VictoriaLogs — естественный выбор |
| Нужна простота и проверенность | Fluent Bit — стандарт де-факто |
| Нужна мощная трансформация | Vector — но overhead выше |
| Нужен OpenTelemetry pipeline | OTel Collector |
| Elastic стек | Filebeat — но самый медленный |

---

## VictoriaLogs — ключевые факты

- Специализированное хранилище логов от создателей VictoriaMetrics
- Совместим с Elasticsearch API (миграция без переписывания запросов)
- Нативная интеграция с Grafana
- Не требует схемы — переваривает любые форматы
- Сжатие значительно эффективнее Loki и ClickHouse
- Поддерживает структурированные и неструктурированные логи

---

## Ссылки

- Бенчмарк: https://victoriametrics.com/blog/log-collectors-benchmark-2026/
- Кейс Lenta tech: https://habr.com/ru/companies/lentatech/articles/1012076/
- vlagent docs: https://docs.victoriametrics.com/victorialogs/vlagent/
- GitHub бенчмарка: https://github.com/VictoriaMetrics/log-collectors-benchmark
