# Мониторинг серверов — железо и виртуалки

> Схема мониторинга физических и виртуальных серверов на базе стека
> VictoriaMetrics + Alertmanager + Grafana + node-exporter.
> Репозиторий: github.com/KoshelevDV/infra-monitoring

---

## Стек (что уже есть)

- **VictoriaMetrics** — хранилище метрик, scrape + alerting rules
- **Alertmanager** — маршрутизация алертов → Telegram
- **Grafana** — дашборды
- **node-exporter** — метрики хоста (CPU, RAM, диски, сеть, systemd)
- **cAdvisor** — Docker-контейнеры

---

## Разделение на группы

```
bare_metal     физические серверы (SMART, hwmon, fan RPM, температуры)
virtual        виртуальные серверы / VPS (только стандартные OS-метрики)
monitoring     хост с самим стеком мониторинга
```

В Ansible inventory группы разделены явно. В scrape.yml каждая группа
получает метку `server_type` для фильтрации алертов.

---

## Что собирать

### Виртуальные серверы

```
node_exporter:
  --collector.systemd
  --collector.processes
  --collector.filesystem.mount-points-exclude=...
  --collector.netstat
  --collector.sockstat
```

### Физические серверы (всё выше +)

```
node_exporter:
  --collector.hwmon      температуры CPU/MB, скорости кулеров
  --collector.cpufreq    частоты ядер
  --collector.rapl       энергопотребление (Intel)
  --collector.nvme       NVMe атрибуты (встроено с 1.6+)

smartctl_exporter (отдельный бинарь, порт 9633):
  SMART для всех дисков, опрос каждые 120s
```

Требует:
- `lm-sensors` + `sensors-detect --auto` для hwmon
- `smartmontools` + sudoers для smartctl_exporter (запускает smartctl от root)

---

## Ansible роли

| Роль | Назначение |
|---|---|
| `node-exporter` | Базовая, VM и bare metal |
| `node-exporter-baremetal` | Расширяет базовую: lm-sensors + hwmon/cpufreq/rapl/nvme коллекторы |
| `smartctl-exporter` | SMART мониторинг дисков, только bare metal |

Плейбуки:
```bash
# bare metal
ansible-playbook -i inventories/production playbooks/bare-metal-monitoring.yml

# только один хост
ansible-playbook -i inventories/production playbooks/bare-metal-monitoring.yml --limit bm-01
```

---

## VictoriaMetrics scrape — job_name по типу сервера

```yaml
- job_name: node_bare_metal
  static_configs:
    - targets: [bm-01:9100, bm-02:9100]
      labels:
        server_type: bare_metal

- job_name: node_virtual
  static_configs:
    - targets: [vps-01:9100, vps-02:9100]
      labels:
        server_type: virtual

- job_name: smartctl
  static_configs:
    - targets: [bm-01:9633, bm-02:9633]
      labels:
        server_type: bare_metal
  scrape_interval: 120s    # SMART запрос медленный
```

---

## Алерты

### Общие (VM + bare metal)

Уже есть в `alerts/infrastructure.yml`:
- HostDown, DiskSpace, CPU, Memory, Swap, NetworkErrors, SystemdServiceFailed

### Только bare metal — `alerts/hardware.yml`

**Температуры:**
- `CPUTemperatureHigh` — > 85°C (warning, 5m)
- `CPUTemperatureCritical` — > 95°C (critical, 2m)
- `SystemTemperatureHigh` — материнская плата / чипсет > 70°C (warning, 10m)

**Кулеры:**
- `FanStopped` — RPM упал до 0 у ранее работавшего вентилятора (critical)
- `FanSpeedLow` — RPM > 0 но < 500 (warning)

**SMART:**
- `SMARTHealthFailed` — `smartctl_device_smart_status == 0` (critical)
- `SMARTReallocatedSectors` — > 0 (warning), > 50 (critical)
- `SMARTPendingSectors` — > 0 (warning)
- `SMARTUncorrectableErrors` — > 0 (critical)
- `NVMeCriticalWarning` — `smartctl_device_nvme_critical_warning_total > 0` (critical)

Алерты отфильтрованы по метке `server_type="bare_metal"` — на VM не срабатывают.

---

## Retention — сколько хранить

VictoriaMetrics флаг: `--retentionPeriod=6` (месяцев).

| Тип данных | Retention | Обоснование |
|---|---|---|
| CPU / RAM / Network | 3 месяца | Capacity planning, тренды |
| Disk I/O latency | 3 месяца | Анализ деградации дисков |
| Температуры / Fan RPM | 6 месяцев | Сезонные паттерны |
| SMART raw values | 12 месяцев | Деградация дисков видна медленно |

SMART — копейки по объёму (сотни временных рядов на хост).

---

## Grafana дашборды

- **Node Exporter Full** (ID 1860) — покрывает всё, включая hwmon при включённом коллекторе
- **SMART disk monitoring** (ID 20204) — статус дисков, SMART-атрибуты

---

## Ключевые метрики — справочник

### hwmon (node_exporter --collector.hwmon)

```promql
# Температура CPU
node_hwmon_temp_celsius{chip=~".*coretemp.*|.*k10temp.*|.*zenpower.*"}

# Скорость кулеров
node_hwmon_fan_rpm

# Напряжения
node_hwmon_in_volts
```

### smartctl_exporter

```promql
# Статус SMART (1=OK, 0=FAIL)
smartctl_device_smart_status

# Reallocated sectors
smartctl_device_attribute{attribute_name="Reallocated_Sector_Ct", value_type="raw"}

# Pending sectors
smartctl_device_attribute{attribute_name="Current_Pending_Sector", value_type="raw"}

# Uncorrectable errors
smartctl_device_attribute{attribute_name="Offline_Uncorrectable", value_type="raw"}

# NVMe critical warning
smartctl_device_nvme_critical_warning_total

# Температура диска через SMART
smartctl_device_attribute{attribute_name="Temperature_Celsius", value_type="raw"}

# Наработка диска (часы)
smartctl_device_power_on_hours_total
```

---

## Ссылки

- node-exporter collectors: https://github.com/prometheus/node_exporter#collectors
- smartctl_exporter: https://github.com/prometheus-community/smartctl_exporter
- Node Exporter Full dashboard: https://grafana.com/grafana/dashboards/1860
