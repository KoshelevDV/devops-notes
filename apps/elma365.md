# ELMA365 On-Premises — документация и эксплуатация

> Источник: https://elma365.com/ru/help/platform/
> Редакция: **Standard Kubernetes** (обновляется как Enterprise через Helm)

---

## Содержание

1. [Архитектура](#архитектура)
2. [Редакции и отличия](#редакции)
3. [Системные требования](#системные-требования)
4. [Версионирование и LTS](#версионирование-и-lts)
5. [Установка](#установка)
6. [Обновление](#обновление) ← главное
7. [Офлайн-обновление](#офлайн-обновление)
8. [Ссылки для скачивания](#ссылки-для-скачивания)
9. [Базы данных](#базы-данных)

---

## Архитектура

**Стек:** Go, NodeJS, Angular, микросервисы в Kubernetes.

| Компонент | Версии | Назначение |
|---|---|---|
| PostgreSQL | 13–18 | Основные данные: пользователи, задачи, процессы, настройки |
| MongoDB | 3.6–8.0 | Неструктурированные данные, чаты, лента |
| RabbitMQ | 3.9.15–3.13 | Шина событий между сервисами |
| Valkey / Redis | 7.2.8+ | Кэширование, не хранит данные долговременно |
| S3 (MinIO) | — | Файловое хранилище |
| Kubernetes | 1.21–1.33 | Оркестрация |
| Helm | 3.14.0+ | Деплой |

Все компоненты могут быть:
- Встроены в чарт `elma365-dbs` (для тестирования / Standard)
- Развёрнуты внешне клиентом (рекомендуется для Enterprise / продакшена)

Безопасность: JWT токены, RBAC, секреты в Kubernetes Secrets.

---

## Редакции

| | Standard KinD | Standard Kubernetes | Enterprise Kubernetes |
|---|---|---|---|
| Кластер k8s | KinD (в Docker) | Настоящий k8s | Настоящий k8s |
| Установка | Shell-скрипт | **Helm** | Helm |
| Обновление | Shell-скрипт `--upgrade` | **Как Enterprise (Helm)** | Helm |
| Масштабирование | Ограничено | Да | Да |
| Лицензий | До 200 | Не ограничено | Не ограничено |

> ⚠️ **Standard Kubernetes обновляется по инструкции Enterprise (через Helm)**, не через shell-скрипт.

---

## Системные требования

### ELMA365 Standard Kubernetes / Enterprise (на 500 пользователей)

**Сервер приложения:**
- CPU: x86-64, 8 vCPU+
- RAM: 16 GB+ (фиксированная, не динамическая!)
- Disk: 100 GB, IOPS записи 2000+
- Network: 1 Gbit/s

Масштабирование: +4 vCPU и +8 GB RAM на каждые 500 пользователей.
Максимум на одном инстансе: ~10 000 пользователей.

**PostgreSQL (на 500 пользователей):**
- CPU: 4 vCPU+, RAM: 8 GB+, Disk: 100 GB, IOPS 2000+
- Версии: 13–18

**MongoDB (на 500 пользователей):**
- CPU: 4 vCPU+, RAM: 8 GB+, Disk: 50 GB, IOPS 2000+
- Версии: 3.6–8.0
- MongoDB 5.0+ требует **AVX** инструкций процессора

**RabbitMQ:**
- CPU: 4 vCPU+, RAM: 8 GB+, Disk: 40 GB, IOPS 2000+

**Важно:** резервное копирование — на отдельном сервере.

**ОС для узлов k8s:**
- Ubuntu Server 24.04 / 22.04 (рекомендуется)
- Debian 11 / 10
- Astra Linux 1.7 (Open) — нужна поддержка вендора
- РЕД ОС 7, 8 — нужна поддержка вендора

**Также поддерживается:** Deckhouse 1.69+

---

## Версионирование и LTS

Формат версии: `YEAR.MAJOR.MINOR` → `2026.4.0`

Мажорные релизы для On-Premises: **X.1, X.4, X.7, X.10**
(X.2, X.3, X.5, X.6... — только для SaaS, для On-Premises недоступны)

### Типы релизов

**Обычный мажорный** — поддержка ~6 месяцев (пока не выйдет следующий за следующим).
**LTS (Long Term Support)** — обычно X.10 — поддержка ~2 года, только критические исправления.

### LTS-расписание

| Версия | Выход | Фиксация LTS | Поддержка до |
|---|---|---|---|
| 2025.4 LTS | Май 2025 | Июль 2025 | Конец июля 2026 |
| 2025.10 LTS | Ноябрь 2025 | Февраль 2026 | Конец февраля 2027 |
| 2026.4 LTS | Май 2026 | Август 2026 | Конец октября 2027 (~1.5 года) |
| 2026.10 LTS | Ноябрь 2026 | Февраль 2027 | Конец февраля 2029 (~2 года) |

**Текущая актуальная LTS:** 2025.10 LTS (поддержка до фев 2027)

**Рекомендация:** для продакшена использовать LTS. Между LTS-версиями можно прыгать напрямую, пропуская обычные мажорные.

---

## Установка

### Шаг 1 — Инфраструктура (если разворачивается внешняя)

Для Standard Kubernetes — нужен готовый k8s кластер с:
- ingress-nginx controller
- coredns
- RBAC
- StorageClass (для PVC)

Встроенные БД через Helm-чарт `elma365-dbs`:
```bash
helm repo add elma365 https://charts.elma365.tech
helm repo update
helm show values elma365/elma365-dbs > values-elma365-dbs.yaml
# Заполнить values-elma365-dbs.yaml
helm install elma365-dbs elma365/elma365-dbs -f values-elma365-dbs.yaml [-n namespace]
```

Компоненты в `values-elma365-dbs.yaml` (включить нужные):
```yaml
global:
  postgresql:
    enabled: true
  mongodb:
    enabled: true
  valkey:
    enabled: true
  rabbitmq:
    enabled: true
  minio:
    enabled: true
  elasticsearch:
    enabled: false  # только для ELMA Bot
```

### Шаг 2 — Получить values-elma365.yaml

```bash
helm repo add elma365 https://charts.elma365.tech
helm repo update

# latest версия
helm show values elma365/elma365 > values-elma365.yaml

# Конкретная версия
helm show values elma365/elma365 --version 2026.1.12 > values-elma365.yaml
```

### Шаг 3 — Заполнить values-elma365.yaml

Обязательные параметры:
```yaml
global:
  host: "elma365.company.ru"        # FQDN или IP
  edition: "standard"               # или "enterprise"

bootstrapCompany:
  email: "admin@company.ru"
  password: "YourPass123"           # только [A-Za-z0-9\-_.]

db:
  psqlUrl: "postgres://user:pass@host:5432/elma365"
  mongoUrl: "mongodb://user:pass@host:27017/elma365"
  vahterMongoUrl: "mongodb://user:pass@host:27017/vahter"
  redisUrl: "redis://host:6379"
  amqpUrl: "amqp://user:pass@host:5672"
  s3:
    method: "path"
    accesskeyid: "minioadmin"
    secretaccesskey: "minioadmin"
    bucket: "elma365"
    backend:
      address: "http://minio:9000"
      region: "us-east-1"
    ssl:
      enabled: false
```

> ⚠️ Недопустимые символы в паролях: `! * ' ( ) ; : @ & = + $ , / ? % # [ ] { }`

### Шаг 4 — Установить

```bash
helm upgrade --install elma365 elma365/elma365 \
  -f values-elma365.yaml \
  --timeout=30m \
  --wait \
  [-n namespace]
```

---

## Обновление

> Это основной раздел. Standard Kubernetes = инструкция Enterprise.

### Шаг 1 — Определить количество шагов

```bash
# Текущая версия
helm show chart elma365/elma365
# или
helm list [-n namespace]

# Доступные версии
helm repo add elma365 https://charts.elma365.tech
helm repo update
helm search repo elma365/elma365           # latest
helm search repo elma365/elma365 --versions # все версии
```

**Правило: нельзя пропускать мажорные версии.**
Мажорные релизы: X.1 → X.4 → X.7 → X.10

| Текущая | Целевая | Этапы |
|---|---|---|
| X.1.N | X.4.N | 1 шаг |
| X.1.N | X.7.N | 2 шага: X.1 → X.4.N → X.7.N |
| X.1.N | X.10.N | 3 шага: X.1 → X.4.N → X.7.N → X.10.N |
| LTS → следующая LTS | — | 1 шаг (можно пропускать обычные мажорные) |

**При каждом промежуточном шаге — устанавливать последний минорный релиз этой мажорной.**

### Шаг 2 — Сохранить конфигурацию

```bash
# Если values-elma365.yaml сохранён — использовать его
# Если утрян — восстановить из кластера:
helm get values elma365 [-n namespace] > values-elma365.yaml
```

> ⚠️ Если обновляете с версии **до 2023.4.30** на **2023.4.30+** — структура values изменилась.
> Нужно получить новый шаблон и перенести параметры вручную:
```bash
helm show values elma365/elma365 > values-elma365-new.yaml
# Перенести параметры из старого в новый вручную
```

### Шаг 3 — Таймауты миграции (для больших БД)

Если данных много — увеличить таймауты в `values-elma365.yaml`:

```yaml
global:
  curlMigrationsMaxTime: 18000    # default: 3000 секунд

deploy:
  appconfig:
    migrateTimeout: "3h"          # default: "1h" (раскомментировать)
```

`migrateTimeout` должен совпадать с `--timeout` в команде helm upgrade.

### Шаг 4 — Выполнить обновление

```bash
# Если нужны промежуточные шаги — для каждой версии:
helm upgrade --install elma365 elma365/elma365 \
  -f values-elma365.yaml \
  --version 2026.4.12 \
  --timeout=30m \
  --wait \
  [-n namespace]

# Финальный шаг — до latest:
helm upgrade --install elma365 elma365/elma365 \
  -f values-elma365.yaml \
  --timeout=30m \
  --wait \
  [-n namespace]
```

Время на каждый шаг при небольшом объёме данных: **10–30 минут**.

---

## Офлайн-обновление

Для закрытого контура (без интернета на сервере).

```bash
# 1. На машине с интернетом — скачать образы и чарты
helm repo add elma365 https://charts.elma365.tech
helm repo update

# Скачать чарты промежуточных версий (если нужны)
helm pull elma365/elma365 --version 2026.4.12

# Скачать latest чарт
helm pull elma365/elma365

# 2. Скопировать elma365-X.Y.Z.tgz на сервер

# 3. На сервере — распаковать каждый чарт в отдельный каталог
mkdir /opt/elma365-2026.4.12
tar -xf elma365-2026.4.12.tgz -C /opt/elma365-2026.4.12 --strip-components=1

# 4. Скопировать values-elma365.yaml в каталог чарта
cp values-elma365.yaml /opt/elma365-2026.4.12/

# 5. Обновить
cd /opt/elma365-2026.4.12
helm upgrade --install elma365 ./elma365 \
  -f values-elma365.yaml \
  --timeout=30m \
  --wait \
  [-n namespace]
```

---

## Ссылки для скачивания

### ELMA365 в Kubernetes (Helm чарты)

```
# Latest
https://dl.elma365.com/onPremise/latest/elma365-latest.tar.gz

# Конкретная версия (например 2026.1.12)
https://dl.elma365.com/onPremise/2026/1/12/elma365-2026.1.12.tar.gz

# Последняя в мажорной (например 2026.1)
https://dl.elma365.com/onPremise/latest/2026/elma365-2026.1-latest.tar.gz

# Latest LTS
https://dl.elma365.com/onPremise/latest/elma365-lts-latest.tar.gz

# Конкретная LTS (например 2025.10.44)
https://dl.elma365.com/onPremise/lts/2025/10/elma365-lts-2025.10.44.tar.gz
```

### ELMA365 в Kubernetes (офлайн, без интернета)

```
# Latest офлайн
https://dl.elma365.com/onPremise/latest/elma365-charts-offline-latest

# Конкретная версия офлайн
https://dl.elma365.com/onPremise/2026/1/12/elma365-charts-offline-2026.1.12

# Latest LTS офлайн
https://dl.elma365.com/onPremise/latest/elma365-charts-offline-lts-latest
```

Helm репозиторий: `https://charts.elma365.tech`

---

## Базы данных

### Встроенный чарт elma365-dbs

Устанавливает все компоненты в k8s. Для продакшена рекомендуется вынести наружу.

```bash
helm show values elma365/elma365-dbs > values-elma365-dbs.yaml
helm install elma365-dbs elma365/elma365-dbs -f values-elma365-dbs.yaml
```

### PostgreSQL кластер (внешний, отказоустойчивый)

Минимум 3 ноды. Рекомендуемый стек:
- **etcd** — распределённое хранилище состояния кластера
- **Patroni** — HA для PostgreSQL, управляет failover через etcd
- **PGBouncer** — connection pooling (опционально)
- **HAProxy** — балансировка подключений

Строка подключения в values-elma365.yaml:
```
postgres://username:password@haproxy-host:5432/elma365
```

### Строки подключения (стандартные для встроенных)

При использовании чарта elma365-dbs со стандартными паролями:
```yaml
db:
  psqlUrl: "postgres://postgres:pgpassword@elma365-dbs-postgresql:5432/elma365"
  mongoUrl: "mongodb://root:mongopassword@elma365-dbs-mongodb:27017/elma365"
  vahterMongoUrl: "mongodb://root:mongopassword@elma365-dbs-mongodb:27017/vahter"
  redisUrl: "redis://:redispassword@elma365-dbs-valkey-master:6379"
  amqpUrl: "amqp://user:password@elma365-dbs-rabbitmq:5672"
```

---

## Полезные команды

```bash
# Текущая версия
helm list [-n namespace]
helm show chart elma365/elma365

# Статус релиза
helm status elma365 [-n namespace]

# Все версии в репо
helm search repo elma365/elma365 --versions

# Восстановить values из кластера
helm get values elma365 [-n namespace] > values-elma365.yaml

# Откат к предыдущей версии (если что-то пошло не так)
helm rollback elma365 [-n namespace]

# Посмотреть историю
helm history elma365 [-n namespace]
```

---

## Ссылки на документацию

- Системные требования: https://elma365.com/ru/help/platform/elma365-enterprise-on-premises.html
- Установка Enterprise/Standard k8s: https://elma365.com/ru/help/platform/installing-elma365-enterprise.html
- Обновление (онлайн): https://elma365.com/ru/help/platform/version-update-enterprise.html
- Обновление (офлайн): https://elma365.com/ru/help/platform/offline-version-update-enterprise.html
- Версионирование и LTS: https://elma365.com/ru/help/platform/version-support-elma365-on-premises.html
- Ссылки для скачивания: https://elma365.com/ru/help/platform/links-for-install-elma365.html
- Встроенные БД (elma365-dbs): https://elma365.com/ru/help/platform/embedded-databases-settings.html
- PostgreSQL кластер: https://elma365.com/ru/help/platform/configure-postgresql.html
- Архитектура: https://elma365.com/ru/help/platform/architecture.html
