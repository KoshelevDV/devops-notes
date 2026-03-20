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

---

## Проверка производительности диска

Медленный диск увеличит задержку запросов в кластере и может нарушить его стабильность.

```bash
apt install fio
mkdir test-data
fio --rw=write --ioengine=sync --fdatasync=1 --directory=test-data --size=100m --bs=2300 --name=mytest
```

В выводе смотреть `fdatasync` — 99.00th процентиль должен быть **меньше 10ms**.  
Значение `2000` = 2ms (хорошо), `15000` = 15ms (плохо).  
Минимальный IOPS записи — по системным требованиям.

---

## Варианты инфраструктуры

| Окружение | Рекомендуемая конфигурация |
|---|---|
| Ознакомление | 1 ВМ, встроенные БД, интернет |
| DEV/TEST | 1–2 ВМ, встроенные/внешние БД |
| PROD | 1 ВМ + 3 ВМ для внешних БД |
| PROD (HA) | Несколько ВМ + отказоустойчивые кластеры всех БД |

**PROD-рекомендация для Enterprise/Standard Kubernetes:**
- Встроенные: RabbitMQ, Valkey/Redis (в k8s через elma365-dbs)
- Внешние: PostgreSQL, MongoDB, MinIO S3
- Проксирование S3 в k8s через S3-Gateway
- Проксирование БД в k8s через DB-Gateway

---

## MinIO / S3 конфигурация

### Установка MinIO (SNSD — Single Node Single Drive)

```bash
# 1. Подготовить диск (XFS рекомендуется)
sudo mkdir -p /var/lib/minio/data1
sudo mkfs.xfs /dev/sdb -L DISK1
echo "LABEL=DISK1 /var/lib/minio/data1 xfs defaults,noatime 0 2" >> /etc/fstab
sudo mount -av

# 2. Установка
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio && sudo mv minio /usr/local/bin/

# 3. MinIO Client
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc && sudo mv mc /usr/local/bin/

# 4. Пользователь
sudo groupadd -r minio-user
sudo useradd -M -r -g minio-user minio-user
sudo chown minio-user:minio-user /var/lib/minio/data1
sudo mkdir -p /etc/minio/certs/CAs
sudo chown -R minio-user:minio-user /etc/minio /var/lib/minio

# 5. Сервис systemd
sudo curl -O https://raw.githubusercontent.com/minio/minio-service/master/linux-systemd/minio.service
sudo mv minio.service /etc/systemd/system
```

**Файл `/etc/default/minio`:**
```ini
MINIO_VOLUMES="/var/lib/minio/data1/minio"
MINIO_OPTS="--certs-dir /etc/minio/certs --console-address :9001"
MINIO_REGION="ru-central-1"
MINIO_ROOT_USER=elma365user
MINIO_ROOT_PASSWORD=SecretPassword
# MINIO_SERVER_URL="https://minio.example:9000"
```

```bash
# 6. Запуск
sudo systemctl daemon-reload
sudo systemctl enable minio.service
sudo systemctl start minio.service
sudo systemctl status minio.service
journalctl -f -u minio.service

# 7. Создать alias
/usr/local/bin/mc alias set minio http://minio.your_domain:9000 elma365user SecretPassword

# 8. Создать бакет (имя должно начинаться с s3elma365)
mc mb minio/s3elma365

# 9. CORS — настраивается через веб-интерфейс MinIO или mc
```

### Подключение MinIO в values-elma365.yaml

```yaml
db:
  s3:
    method: "path"
    accesskeyid: "elma365user"
    secretaccesskey: "SecretPassword"
    bucket: "s3elma365"
    backend:
      address: "http://minio.your_domain:9000"
      region: "ru-central-1"
    ssl:
      enabled: false
```

### TLS/SSL для MinIO

Положить в `/etc/minio/certs/`:
- `public.crt` — сертификат сервера
- `private.key` — закрытый ключ
- `CAs/` — корневой CA при самоподписанном

---

## MongoDB кластер (внешний)

### Установка MongoDB 6.0 на Ubuntu 22.04

```bash
# На каждой ноде (минимум 3)
sudo apt-get install gnupg
curl -fsSL https://pgp.mongodb.com/server-6.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-6.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
sudo apt-get update
sudo apt install mongodb-org
sudo systemctl enable --now mongod
```

### Настройка (на mongodb-server1)

```bash
mongosh
use elma365
db.createUser({user:'elma365', pwd:'SecretPassword', roles:[{role:"readWrite", db:"elma365"},{"role":"root","db":"admin"}]})
use admin
db.createUser({user:'superuser', pwd:'SecretPassword', roles: ["root"]})
exit
```

### `/etc/mongod.conf` на каждой ноде

```yaml
net:
  port: 27017
  bindIp: 0.0.0.0

replication:
  replSetName: "rs0"
  enableMajorityReadConcern: true
```

```bash
sudo systemctl restart mongod
```

### Инициализация реплики (на mongodb-server1)

```bash
sudo mongosh -u superuser admin
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongodb-server1.your_domain" },
    { _id: 1, host: "mongodb-server2.your_domain" },
    { _id: 2, host: "mongodb-server3.your_domain" }
  ]
})
rs.conf()
```

### Безопасность MongoDB

```bash
openssl rand -base64 756 > /var/lib/mongodb/keyfile
chmod 400 /var/lib/mongodb/keyfile
chown mongodb:mongodb /var/lib/mongodb/keyfile
# Скопировать на все ноды с теми же правами
```

`/etc/mongod.conf` добавить:
```yaml
setParameter:
  enableLocalhostAuthBypass: false
security:
  authorization: "enabled"
  keyFile: /var/lib/mongodb/keyfile
```

### Строка подключения MongoDB

```
mongodb://elma365:SecretPassword@mongodb-server1.your_domain:27017,mongodb-server2.your_domain:27017,mongodb-server3.your_domain:27017/elma365?replicaSet=rs0&readPreference=nearest&maxStalenessSeconds=120
```

### TLS/SSL для MongoDB

```yaml
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /path/to/mongodb.pem  # cat key > pem; cat fullchain >> pem
```

---

## RabbitMQ кластер (внешний)

> ⚠️ **RabbitMQ 4.0 не поддерживается.** Использовать 3.9.15–3.13.

### Установка RabbitMQ 3.12.0 + Erlang 25.3.2.2 на Ubuntu 20.04/22.04

```bash
# Ключи и репозиторий (упрощённо)
sudo apt-get install curl gnupg apt-transport-https -y
# Добавить ключи и репозитории (см. официальную документацию)

sudo apt-get update -y

# Erlang
sudo apt-get install -y \
  erlang-base=1:25.3.2.2-1 erlang-asn1=1:25.3.2.2-1 erlang-crypto=1:25.3.2.2-1 \
  erlang-eldap=1:25.3.2.2-1 erlang-ssl=1:25.3.2.2-1 erlang-mnesia=1:25.3.2.2-1 \
  erlang-os-mon=1:25.3.2.2-1 erlang-tools=1:25.3.2.2-1

# RabbitMQ
sudo apt-get install rabbitmq-server=3.12.0-1 -y --fix-missing
sudo systemctl enable --now rabbitmq-server
```

### Кластеризация

```bash
# На каждой ноде создать /etc/rabbitmq/rabbitmq-env.conf:
RABBITMQ_NODENAME=rabbit@rabbitmq-server1.your_domain
RABBITMQ_USE_LONGNAME=true

# Скопировать /var/lib/rabbitmq/.erlang.cookie с server1 на все остальные (одинаковый!)
sudo systemctl restart rabbitmq-server

# На server2 и server3:
sudo rabbitmqctl stop_app
sudo rabbitmqctl reset
sudo rabbitmqctl join_cluster rabbit@rabbitmq-server1.your_domain
sudo rabbitmqctl start_app
sudo rabbitmqctl cluster_status
```

### Настройка пользователя и vhost (на server1)

```bash
sudo rabbitmq-plugins enable rabbitmq_management
sudo rabbitmqctl add_vhost elma365vhost
sudo rabbitmqctl add_user elma365user SecretPassword
sudo rabbitmqctl set_permissions -p elma365vhost elma365user ".*" ".*" ".*"
sudo rabbitmqctl set_user_tags elma365user administrator

# Политика HA (зеркалирование)
sudo rabbitmqctl set_policy -p 'elma365vhost' MirrorAllQueues ".*" '{"ha-mode":"all"}'
# Или Quorum если HA не поддерживается:
# rabbitmqctl set_policy –p 'elma365vhost' QuorumDefault "^.*" '{"queue-type":"quorum"}' --priority 0 --apply-to queues
```

### Строка подключения RabbitMQ

```
amqp://elma365user:SecretPassword@haproxy-host:5672/elma365vhost
```

---

## Redis / Valkey кластер (внешний)

> ⚠️ Redis меняет лицензию — рекомендуется Valkey как альтернатива (см. статью о Valkey cluster).

### Установка Redis 6.2.12 на Ubuntu 20.04/22.04

```bash
sudo apt install lsb-release curl gpg
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
sudo apt-get update
sudo apt-get -y install redis=6:6.2.12-1rl1~$(lsb_release -cs)1 redis-server=6:6.2.12-1rl1~$(lsb_release -cs)1 redis-tools=6:6.2.12-1rl1~$(lsb_release -cs)1 redis-sentinel=6:6.2.12-1rl1~$(lsb_release -cs)1
```

### `/etc/redis/redis.conf` на каждой ноде

```ini
bind 0.0.0.0
maxclients 20000
maxmemory-policy allkeys-lfu
save ""
appendonly no
masterauth SecretPassword
requirepass SecretPassword
# На каждой ноде указать свой FQDN:
replica-announce-ip redis-server1.your_domain
# На server2 и server3 добавить:
replicaof redis-server1.your_domain 6379
```

```bash
sudo systemctl restart redis-server
sudo systemctl enable redis-server
# Проверка на мастере:
sudo redis-cli -a SecretPassword info replication
```

### `/etc/redis/sentinel.conf` на каждой ноде

```ini
bind 0.0.0.0
sentinel announce-ip redis-server1.your_domain  # свой FQDN
sentinel monitor mymaster redis-server1.your_domain 6379 2
sentinel auth-pass mymaster SecretPassword
sentinel down-after-milliseconds mymaster 3000
sentinel failover-timeout mymaster 6000
sentinel resolve-hostnames yes
sentinel announce-hostnames yes
user default on >SecretPassword sanitize-payload ~* &* +@all
```

```bash
sudo systemctl restart redis-sentinel
sudo systemctl enable redis-sentinel
sudo redis-cli -p 26379 info sentinel
```

### Строка подключения Redis (Sentinel)

```
redis-sentinel://SecretPassword@redis-server1.your_domain:26379,redis-server2.your_domain:26379,redis-server3.your_domain:26379?sentinelMasterId=mymaster
```

Или для Standalone (не Sentinel):
```
redis://:SecretPassword@redis-server1.your_domain:6379
```

---

## PostgreSQL — внешний (standalone для Standard KinD / PROD)

### Установка PostgreSQL 18 на Ubuntu 24.04

```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt update
sudo apt install postgresql-18
```

### Настройка

```bash
# Создать роль и БД
sudo -u postgres psql -c "CREATE ROLE elma365 WITH login password 'SecretPassword';"
sudo -u postgres psql -c "CREATE DATABASE elma365 WITH OWNER elma365;"

# Расширения
sudo -u postgres psql -d elma365 -c "CREATE EXTENSION \"uuid-ossp\"; CREATE EXTENSION pg_trgm;"
```

**`/etc/postgresql/18/main/postgresql.conf`:**
```ini
listen_addresses = 'localhost, 192.168.10.10'
max_connections = 2000
max_locks_per_transaction = 512
standard_conforming_strings = on
timezone = 'UTC'
```

**`/etc/postgresql/18/main/pg_hba.conf` — в конце:**
```
host    all             all             192.168.0.0/16              md5
```

```bash
sudo systemctl restart postgresql
```

### TLS/SSL для PostgreSQL

```ini
ssl = on
ssl_ca_file = '/etc/ssl/certs/CA.pem'
ssl_cert_file = '/etc/ssl/certs/ssl-cert.pem'
ssl_key_file = '/etc/ssl/private/server.key'
```

### Строка подключения PostgreSQL

```
postgresql://elma365:SecretPassword@<postgresql-server>:5432/elma365?sslmode=disable
# С TLS:
postgresql://elma365:SecretPassword@<postgresql-server>:5432/elma365?sslmode=require
# Через PGBouncer (порт 6432):
postgresql://elma365:SecretPassword@<postgresql-server>:6432/elma365?sslmode=disable
```

### Создание БД с локалью ru_RU.UTF-8 (опционально)

```bash
sudo locale-gen ru_RU.UTF-8 && sudo update-locale
sudo systemctl restart postgresql
sudo -u postgres psql -c "CREATE DATABASE elma365 WITH OWNER = elma365
  LOCALE_PROVIDER = icu ICU_LOCALE = 'ru-RU' ENCODING = 'UTF8'
  LC_COLLATE = 'ru_RU.UTF-8' LC_CTYPE = 'ru_RU.UTF-8' TEMPLATE = template0;"
```

---

## Лицензирование

### Типы лицензий

| Тип | Описание |
|---|---|
| **Именная** | Привязана к конкретному пользователю. 1 лицензия = 1 аккаунт (занята всегда, даже офлайн) |
| **Конкурентная** | Не привязана к пользователю. Определяет число одновременных сессий. Удобна для периодических пользователей |
| **Серверная** | Активирует системное решение для всех пользователей платформы |
| **Именная внешних пользователей** | Для портала — одна лицензия = одна учётная запись внешнего пользователя |

**Standard KinD:** до 200 пользователей  
**Standard Kubernetes / Enterprise:** без ограничений

### Комбинирование лицензий

Можно смешивать именные и конкурентные. Сотрудники с гарантированным доступом → добавить в группу **Привилегированные пользователи** (получат именные лицензии).

### Управление лицензиями

Администрирование → Управление лицензиями → Лицензия платформы.

Заблокированный пользователь **освобождает** именную лицензию.

Для конкурентных лицензий можно настроить таймаут бездействующей сессии (автоосвобождение).

### Системные решения

- **ECM, Advanced Security Pack** — активируются без лицензии
- **CRM, Omni, Проекты** — именные / конкурентные лицензии
- **Полнотекстовый поиск** — серверная лицензия
- **ELMA Bot** — серверная лицензия, приобретается отдельно

---

## Изменение параметров после установки

### ELMA365 Standard KinD — через `config-elma365.txt`

Файл создаётся при первом запуске установочного скрипта. После изменений — применять:

```bash
sudo ./elma365-docker.sh --upgrade
```

**Основные параметры config-elma365.txt:**

| Параметр | Описание |
|---|---|
| `ELMA365_HOST` | IP или FQDN (в DNS нужна A-запись) |
| `ELMA365_EMAIL` | Email супервизора (только при первой установке) |
| `ELMA365_PASSWORD` | Пароль супервизора |
| `ELMA365_LANGUAGE` | Язык: `ru-RU`, `en-US`, `sk-SK` |
| `ELMA365_EDITION` | `standard` или `enterprise` |
| `ELMA365_DB_PSQL` | Строка подключения к PostgreSQL |
| `ELMA365_DB_PSQL_RO` | Строка подключения к PostgreSQL (только чтение) |
| `ELMA365_DB_MONGO` | Строка подключения к MongoDB |
| `ELMA365_DB_S3_ADDRESS` | Адрес MinIO/S3 |
| `ELMA365_DB_S3_BUCKET` | Имя бакета (формат `s3elma365*`) |
| `ELMA365_DB_S3_USER` | Пользователь S3 |
| `ELMA365_DB_S3_PASSWORD` | Пароль S3 |

**SMTP-параметры:**

| Параметр | Описание |
|---|---|
| `ELMA365_SMTP_HOST` | IP/URL SMTP-сервера |
| `ELMA365_SMTP_PORT` | Порт SMTP |
| `ELMA365_SMTP_FROM` | Email отправителя |
| `ELMA365_SMTP_USER` | Логин SMTP |
| `ELMA365_SMTP_PASSWORD` | Пароль SMTP |
| `ELMA365_SMTP_TLS` | `false` (если используется ELMA365_MAILER_SMTPTLS) |
| `ELMA365_MAILER_SMTPTLS` | С версии 2025.10.12: `notls`, `tls`, `starttls` |

**TLS/HTTPS-параметры:**

| Параметр | Описание |
|---|---|
| `ELMA365_TLS_CRT` | Путь к fullchain SSL-сертификату |
| `ELMA365_TLS_KEY` | Путь к закрытому ключу |
| `ELMA365_TLS_CA` | Путь к корневому CA (для самоподписанных) |
| `ELMA365_CERTMANAGER` | `true` — автоматический Let's Encrypt (нужен порт 80) |
| `ELMA365_CERTMANAGER_SELFSIGNED` | `true` — самоподписанный через cert-manager |

> ⚠️ Параметры TLS игнорируются если в `ELMA365_HOST` указан IP-адрес.

### ELMA365 Enterprise/Standard Kubernetes — через values-elma365.yaml + helm upgrade

```bash
helm upgrade --install elma365 elma365/elma365 \
  -f values-elma365.yaml \
  --timeout=30m \
  --wait \
  [-n namespace]
```

---

## Офлайн установка (air-gap)

### ELMA365 Standard KinD — офлайн

```bash
# 1. На машине с интернетом:
sudo curl -fsSL -o elma365-docker.sh https://dl.elma365.com/onPremise/latest/elma365-docker-offline-latest
chmod +x elma365-docker.sh
./elma365-docker.sh   # скачивает ~4-5 ГБ в каталог elma365-X.Y.Z

# 2. Скопировать каталог elma365-X.Y.Z на изолированный сервер

# 3. На изолированном сервере установить Docker (KinD не поддерживает Cgroups v2!)

# 4. Создать конфиг (первый запуск):
sudo ./elma365-docker.sh --offline
# Заполнить config-elma365.txt

# 5. Запустить установку
sudo ./elma365-docker.sh --offline
```

> ⚠️ KinD не поддерживает Cgroups v2. При проблемах смотреть документацию KinD known-issues.

### ELMA365 Enterprise/Standard Kubernetes — офлайн (Harbor + скрипт)

**Шаг 1. Создать проект в Harbor (Projects → +New project, например `images`)**

**Шаг 2. Скачать образы на машине с интернетом:**

```bash
curl -fsSL -o elma365-charts-offline.sh https://dl.elma365.com/onPremise/latest/elma365-charts-offline-latest
chmod +x elma365-charts-offline.sh

# Интерактивно (выбрать pull + пакеты):
./elma365-charts-offline.sh

# Или скачать всё:
./elma365-charts-offline.sh --pull

# Скопировать каталог elma365-X.Y.Z на изолированный сервер
```

**Шаг 3. Загрузить образы в Harbor:**

```bash
cd elma365-X.Y.Z
./elma365-charts-offline.sh  # выбрать push + нужные пакеты
# Или неинтерактивно:
./elma365-charts-offline.sh --push
```

**Ссылки для офлайн-скриптов:**
```
# Latest офлайн
https://dl.elma365.com/onPremise/latest/elma365-charts-offline-latest
https://dl.elma365.com/onPremise/latest/elma365-docker-offline-latest

# Конкретная версия
https://dl.elma365.com/onPremise/2026/1/12/elma365-charts-offline-2026.1.12
```

---

## Deckhouse-специфика (air-gap кластер)

Deckhouse — рекомендуемая платформа k8s для ELMA365. Включена в реестр российского ПО, сертифицирована CNCF.

### Требования для установки

**Персональный компьютер (инсталлятор):**
- ОС: Windows 10+, macOS 10.15+, Ubuntu 18.04+, Fedora 35+
- Установленный Docker
- SSH-доступ по ключу к master-узлу

> ⚠️ Нельзя запускать инсталлятор на том же узле, где будет master! Docker не должен быть установлен на master-узле до установки Deckhouse.

**Master-узел:**
- ≥ 12 CPU, ≥ 16 GB RAM, ≥ 200 GB диск
- Поддерживаемая ОС
- Доступ к registry (через прокси или локальный Harbor)
- Без установленного container runtime (containerd, docker)

### Этапы установки

1. Скачать образы Deckhouse → загрузить в локальный реестр (Harbor)
2. Сгенерировать `config.yml` через сервис [Быстрый старт Deckhouse](https://deckhouse.ru/gs/)
3. Запустить инсталлятор Deckhouse в Docker с персонального компьютера

### Пример `config.yml` (статический кластер, интернет)

```yaml
apiVersion: deckhouse.io/v1
kind: ClusterConfiguration
clusterType: Static
podSubnetCIDR: 10.111.0.0/16
serviceSubnetCIDR: 10.222.0.0/16
kubernetesVersion: "Automatic"
clusterDomain: "cluster.local"
---
apiVersion: deckhouse.io/v1
kind: InitConfiguration
deckhouse:
  imagesRepo: registry.deckhouse.ru/deckhouse/ce
  registryDockerCfg: eyJhdXRocyI6...
---
apiVersion: deckhouse.io/v1alpha1
kind: ModuleConfig
metadata:
  name: global
spec:
  version: 2
  settings:
    modules:
      publicDomainTemplate: "%s.elewise.local"
---
apiVersion: deckhouse.io/v1alpha1
kind: ModuleConfig
metadata:
  name: cni-cilium
spec:
  version: 1
  enabled: true
  settings:
    tunnelMode: VXLAN
```

### Ключевые параметры конфига

- `podSubnetCIDR` — сеть подов
- `serviceSubnetCIDR` — сеть сервисов
- `kubernetesVersion` — версия k8s (`"Automatic"` = последняя в канале)
- `releaseChannel` — канал обновлений (`EarlyAccess` рекомендуется)
- `publicDomainTemplate` — шаблон для системных доменов (Grafana, etc.)
- `podNetworkMode` — режим flannel: `VXLAN` (L3) или `HostGW` (L2)
- `internalNetworkCIDRs` — внутренняя сеть узлов кластера

---

## ELMA Bot

ELMA Bot — low-code конструктор чат-ботов на ML. Работает с ELMA365 CRM и ELMA365 Omni.

**Режимы работы:**
- **Чат-бот** — ведёт диалог вместо оператора
- **Суфлёр** — подсказывает ответы оператору в реальном времени

**Особенности:**
- ML-обучение автоматически запускается при изменении сценариев и триггеров
- Логика — сценарии + база знаний
- К одной линии можно подключить только 1 чат-бот и 1 суфлёра

**Лицензирование:** серверная лицензия, покупается отдельно.  
В On-Premises и SaaS Enterprise — требуется предварительная настройка интеграции.

**Требует:** отдельный компонент Elasticsearch (включить в `elma365-dbs` `values.yaml`:  `elasticsearch.enabled: true`)

---

## Полезные диагностические команды

```bash
# Helm — статус и история
helm list [-n namespace]
helm status elma365 [-n namespace]
helm history elma365 [-n namespace]

# Посмотреть текущие values
helm get values elma365 [-n namespace]

# Откат
helm rollback elma365 [revision] [-n namespace]

# Kubernetes — поды ELMA365
kubectl get pods [-n namespace]
kubectl describe pod <pod-name> [-n namespace]
kubectl logs <pod-name> [-n namespace] --tail=100

# MinIO — проверка
systemctl status minio.service
journalctl -u minio.service -n 50

# MongoDB — проверка реплики
mongosh -u superuser admin
rs.status()

# RabbitMQ — кластер
rabbitmqctl cluster_status
rabbitmqctl list_policies --vhost elma365vhost

# Redis — репликация
redis-cli -a SecretPassword info replication
redis-cli -p 26379 info sentinel

# PostgreSQL — подключения
sudo -u postgres psql -c "SELECT count(*) FROM pg_stat_activity;"
```

