# PostgreSQL: Бэкап, Восстановление, Миграция, Синхронизация

## Содержание
1. [Утилиты — обзор](#утилиты--обзор)
2. [pg_dump / pg_restore](#pg_dump--pg_restore)
3. [pg_dumpall](#pg_dumpall)
4. [pg_basebackup](#pg_basebackup)
5. [PITR — Point-in-Time Recovery](#pitr--point-in-time-recovery)
6. [Перенос БД между инстансами](#перенос-бд-между-инстансами)
7. [Синхронизация и переливка данных](#синхронизация-и-переливка-данных)
8. [Сторонние инструменты бэкапа](#сторонние-инструменты-бэкапа)
9. [CDC и Kafka / Debezium](#cdc-и-kafka--debezium)
10. [UI-инструменты](#ui-инструменты)
11. [Рекомендации и подводные камни](#рекомендации-и-подводные-камни)

---

## Утилиты — обзор

| Утилита | Что делает | Когда использовать |
|---------|-----------|-------------------|
| `pg_dump` | Дамп одной БД (логический) | Перенос, бэкап одной БД |
| `pg_dumpall` | Дамп всего кластера + роли | Полная миграция инстанса |
| `pg_restore` | Восстановление из бинарного дампа | С форматами `-Fc`, `-Fd`, `-Ft` |
| `pg_basebackup` | Физическая копия кластера | Реплики, быстрый полный бэкап |
| `psql` | Восстановление из SQL-дампа | Формат `-Fp` (plain text) |
| `pglogical` | Логическая репликация | Online-синхронизация между инстансами |
| `pg_logical` | Встроенная логическая репликация | PostgreSQL 10+ |
| `COPY / \copy` | Экспорт/импорт таблиц в CSV | Переливка данных между таблицами |
| `postgres_fdw` | Foreign Data Wrapper | Запросы к удалённой БД напрямую |

---

## pg_dump / pg_restore

### Форматы дампа

| Флаг | Формат | Восстановление |
|------|--------|---------------|
| `-Fp` | Plain SQL (по умолчанию) | `psql -f dump.sql` |
| `-Fc` | Custom (бинарный, сжатый) | `pg_restore` |
| `-Fd` | Directory (параллельный) | `pg_restore -j N` |
| `-Ft` | Tar | `pg_restore` |

Рекомендуется `-Fc` или `-Fd` — сжатие + возможность параллельного восстановления.

### Бэкап одной БД

```bash
# Полный дамп
pg_dump -h localhost -U postgres -d mydb -Fc -f mydb.dump

# Только схема
pg_dump -h localhost -U postgres -d mydb --schema-only -Fc -f mydb_schema.dump

# Только данные
pg_dump -h localhost -U postgres -d mydb --data-only -Fc -f mydb_data.dump

# Без прав и владельцев (для переноса на другой инстанс)
pg_dump -h localhost -U postgres -d mydb \
  --no-owner --no-privileges \
  -Fc -f mydb.dump

# Конкретная схема
pg_dump -h localhost -U postgres -d mydb -n myschema -Fc -f myschema.dump

# Конкретные таблицы
pg_dump -h localhost -U postgres -d mydb \
  -t users -t orders \
  -Fc -f tables.dump

# Исключить таблицы
pg_dump -h localhost -U postgres -d mydb \
  -T logs -T audit_log \
  -Fc -f mydb_no_logs.dump
```

### Восстановление

```bash
# Создать БД перед восстановлением
psql -h localhost -U postgres -c "CREATE DATABASE mydb;"

# Восстановить из custom-дампа
pg_restore -h localhost -U postgres -d mydb mydb.dump

# Параллельное восстановление (быстрее на больших БД)
pg_restore -h localhost -U postgres -d mydb -j 4 mydb.dump

# Без ошибок при конфликтах (--clean удаляет объекты перед созданием)
pg_restore -h localhost -U postgres -d mydb --clean --if-exists mydb.dump

# Только схема
pg_restore -h localhost -U postgres -d mydb --schema-only mydb.dump

# Только данные
pg_restore -h localhost -U postgres -d mydb --data-only mydb.dump

# Восстановить plain SQL дамп
psql -h localhost -U postgres -d mydb -f mydb.sql
```

### Pipe — без промежуточного файла

```bash
pg_dump -h source -U postgres -d mydb -Fc --no-owner --no-privileges \
  | pg_restore -h target -U postgres -d mydb --no-owner --no-privileges -j 4
```

---

## pg_dumpall

Дампит весь кластер: все БД + глобальные объекты (роли, табличные пространства).

```bash
# Полный дамп кластера
pg_dumpall -h localhost -U postgres -f full_cluster.sql

# Только глобальные объекты (роли, tablespaces) — без данных БД
pg_dumpall -h localhost -U postgres --globals-only -f globals.sql

# Восстановление
psql -h target -U postgres -f full_cluster.sql
```

> ⚠️ Формат только plain SQL — параллельное восстановление недоступно. На больших кластерах медленно.

---

## pg_basebackup

Физическая копия кластера на уровне файлов. Самый быстрый способ создать реплику или полный бэкап.

```bash
# Бэкап в директорию
pg_basebackup -h localhost -U replicator -D /backup/pgbase -Fp -Xs -P

# Бэкап в tar + сжатие
pg_basebackup -h localhost -U replicator -D /backup/pgbase -Ft -z -P

# Флаги:
# -Fp / -Ft  — формат (plain / tar)
# -Xs        — включить WAL-сегменты в бэкап (stream)
# -P         — прогресс
# -R         — создать standby.signal + postgresql.auto.conf (для реплики)

# Создать реплику сразу
pg_basebackup -h primary -U replicator -D /var/lib/postgresql/17/main \
  -Fp -Xs -P -R
```

> Требуется пользователь с ролью `REPLICATION` или суперпользователь.
> На источнике: `wal_level = replica` в `postgresql.conf`.

---

## PITR — Point-in-Time Recovery

Позволяет восстановить БД на любой момент времени по WAL-логам.

### Настройка архивирования WAL

```conf
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'cp %p /archive/wal/%f'  # или rsync, AWS S3 и т.д.
```

### Восстановление на момент времени

```conf
# recovery.conf (PG < 12) или postgresql.conf (PG 12+)
restore_command = 'cp /archive/wal/%f %p'
recovery_target_time = '2026-02-25 12:00:00'
recovery_target_action = 'promote'  # или 'pause'
```

```bash
# Создать файл-триггер
touch /var/lib/postgresql/17/main/recovery.signal  # PG 12+
```

> Для продакшена — использовать **pgBackRest** или **Barman** вместо ручного PITR.

---

## Перенос БД между инстансами

### Сценарий 1: Одна БД, одинаковые версии

```bash
# Быстро через pipe
pg_dump -h source -U postgres -d mydb -Fc \
  --no-owner --no-privileges \
  | pg_restore -h target -U postgres -d mydb \
    --no-owner --no-privileges -j 4
```

### Сценарий 2: Перенос всего кластера (одинаковые версии)

```bash
# Остановить источник (или взять basebackup на живом)
pg_basebackup -h source -U replicator \
  -D /var/lib/postgresql/17/main -Fp -Xs -P

# Запустить целевой инстанс
```

### Сценарий 3: Смена мажорной версии (например, 14 → 17)

```bash
# pg_upgrade — самый быстрый способ
pg_upgrade \
  -b /usr/lib/postgresql/14/bin \
  -B /usr/lib/postgresql/17/bin \
  -d /var/lib/postgresql/14/main \
  -D /var/lib/postgresql/17/main \
  -j 4        # параллельно
  # --link    # hard link вместо копирования (быстрее, но источник нельзя трогать)
```

> После `pg_upgrade` запустить `vacuumdb --all --analyze-in-stages` для обновления статистики.

### Сценарий 4: Минимальный даунтайм (логическая репликация)

1. Создать публикацию на источнике:
```sql
CREATE PUBLICATION mypub FOR ALL TABLES;
```

2. Создать подписку на целевом:
```sql
CREATE SUBSCRIPTION mysub
  CONNECTION 'host=source dbname=mydb user=replicator password=...'
  PUBLICATION mypub;
```

3. Дождаться синхронизации, переключить приложение, удалить подписку.

---

## Синхронизация и переливка данных

### COPY — экспорт/импорт таблиц

```bash
# Экспорт в CSV (серверная сторона, нужны права superuser)
psql -h source -U postgres -d mydb \
  -c "COPY users TO '/tmp/users.csv' CSV HEADER;"

# Клиентская сторона (без прав superuser)
psql -h source -U postgres -d mydb \
  -c "\copy users TO '/local/users.csv' CSV HEADER"

# Импорт
psql -h target -U postgres -d mydb \
  -c "\copy users FROM '/local/users.csv' CSV HEADER"

# Фильтрация при экспорте
psql -h source -U postgres -d mydb \
  -c "\copy (SELECT * FROM users WHERE created_at > '2026-01-01') TO '/tmp/users_2026.csv' CSV HEADER"
```

### postgres_fdw — прямые запросы к удалённой БД

```sql
-- Установить расширение
CREATE EXTENSION postgres_fdw;

-- Создать внешний сервер
CREATE SERVER remote_db
  FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (host 'source_host', port '5432', dbname 'mydb');

-- Маппинг пользователя
CREATE USER MAPPING FOR current_user
  SERVER remote_db
  OPTIONS (user 'postgres', password 'secret');

-- Импортировать схему
IMPORT FOREIGN SCHEMA public
  FROM SERVER remote_db
  INTO fdw_schema;

-- Переливка данных
INSERT INTO local_users SELECT * FROM fdw_schema.users;

-- Или синхронизация (upsert)
INSERT INTO local_users
  SELECT * FROM fdw_schema.users
ON CONFLICT (id) DO UPDATE SET
  name = EXCLUDED.name,
  updated_at = EXCLUDED.updated_at;
```

### Логическая репликация (встроенная, PG 10+)

Подходит для онлайн-синхронизации без даунтайма.

```sql
-- Источник: включить в postgresql.conf
-- wal_level = logical

-- Источник: создать публикацию
CREATE PUBLICATION sync_pub FOR TABLE users, orders;
-- Или все таблицы:
CREATE PUBLICATION sync_pub FOR ALL TABLES;

-- Приёмник: создать подписку
CREATE SUBSCRIPTION sync_sub
  CONNECTION 'host=source port=5432 dbname=mydb user=replicator password=...'
  PUBLICATION sync_pub;

-- Статус репликации
SELECT * FROM pg_stat_subscription;
SELECT * FROM pg_replication_slots;  -- на источнике

-- Остановить синхронизацию
ALTER SUBSCRIPTION sync_sub DISABLE;
DROP SUBSCRIPTION sync_sub;

-- Удалить слот на источнике (важно! иначе WAL будет накапливаться)
SELECT pg_drop_replication_slot('sync_sub');
```

> ⚠️ Логическая репликация не переносит DDL (ALTER TABLE и т.д.) — только DML. Схему нужно синхронизировать отдельно.

### pglogical (расширение, больше возможностей)

```bash
# Установка
apt install postgresql-17-pglogical

# postgresql.conf
# shared_preload_libraries = 'pglogical'
# wal_level = logical
```

```sql
-- Источник
CREATE EXTENSION pglogical;
SELECT pglogical.create_node('provider', 'host=source dbname=mydb');
SELECT pglogical.create_replication_set('myset');
SELECT pglogical.replication_set_add_all_tables('myset', ARRAY['public']);

-- Приёмник
CREATE EXTENSION pglogical;
SELECT pglogical.create_node('subscriber', 'host=target dbname=mydb');
SELECT pglogical.create_subscription('mysub',
  'host=source dbname=mydb user=replicator',
  ARRAY['myset']);
```

---

## Сторонние инструменты бэкапа

### pgBackRest

Де-факто стандарт для продакшн-бэкапов PostgreSQL. Инкрементальные бэкапы, параллельность, шифрование, поддержка S3/GCS/Azure.

```bash
# Установка
apt install pgbackrest

# /etc/pgbackrest/pgbackrest.conf
[global]
repo1-path=/backup/pgbackrest
repo1-retention-full=2
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=secretpassword

[mydb]
pg1-path=/var/lib/postgresql/17/main

# Инициализация репозитория
pgbackrest --stanza=mydb stanza-create

# Полный бэкап
pgbackrest --stanza=mydb --type=full backup

# Инкрементальный
pgbackrest --stanza=mydb --type=incr backup

# Дифференциальный (от последнего full)
pgbackrest --stanza=mydb --type=diff backup

# Список бэкапов
pgbackrest --stanza=mydb info

# Восстановление
systemctl stop postgresql
pgbackrest --stanza=mydb restore

# PITR — восстановление на момент времени
pgbackrest --stanza=mydb restore \
  --target="2026-02-25 12:00:00" \
  --target-action=promote

# Бэкап в S3
# repo1-type=s3
# repo1-s3-bucket=my-bucket
# repo1-s3-endpoint=s3.amazonaws.com
# repo1-s3-region=us-east-1
# repo1-s3-key=AKIAIOSFODNN7
# repo1-s3-key-secret=secret
```

**UI:** нет встроенного. Сторонние:
- [pgbackrest-exporter](https://github.com/woblerr/pgbackrest_exporter) — Prometheus-экспортер → Grafana

---

### Barman

Backup and Recovery Manager от 2ndQuadrant (EnterpriseDB). Централизованный сервер бэкапов для нескольких PostgreSQL-инстансов.

```bash
# Установка
apt install barman barman-cli

# /etc/barman/barman.conf
[barman]
barman_home = /var/lib/barman
barman_user = barman
log_file = /var/log/barman/barman.log

# /etc/barman/conf.d/myserver.conf
[myserver]
description = "My PostgreSQL Server"
conninfo = host=pghost user=barman dbname=postgres
backup_method = postgres        # или rsync
streaming_conninfo = host=pghost user=streaming_barman
streaming_archiver = on
backup_compression = gzip
retention_policy = RECOVERY WINDOW OF 7 DAYS

# Проверка конфигурации
barman check myserver

# Полный бэкап
barman backup myserver

# Список бэкапов
barman list-backups myserver

# Восстановление
barman recover myserver latest /var/lib/postgresql/17/main

# PITR
barman recover myserver latest /var/lib/postgresql/17/main \
  --target-time "2026-02-25 12:00:00"
```

**UI:** нет встроенного. Управление через CLI. Метрики — через `barman-exporter` → Grafana.

---

### WAL-G

Минималистичный и быстрый инструмент для WAL-архивирования и бэкапов в облако (S3, GCS, Azure, Swift). Написан на Go.

```bash
# Установка
curl -L https://github.com/wal-g/wal-g/releases/latest/download/wal-g-pg-ubuntu-20.04-amd64 \
  -o /usr/local/bin/wal-g && chmod +x /usr/local/bin/wal-g

# Переменные окружения (или .walg.json)
export WALG_S3_PREFIX=s3://my-bucket/pg-backups
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7
export AWS_SECRET_ACCESS_KEY=secret
export AWS_REGION=us-east-1
export PGHOST=localhost
export PGUSER=postgres

# postgresql.conf — архивирование WAL через wal-g
archive_mode = on
archive_command = 'wal-g wal-push %p'
restore_command = 'wal-g wal-fetch %f %p'

# Полный бэкап
wal-g backup-push /var/lib/postgresql/17/main

# Список бэкапов
wal-g backup-list

# Восстановление последнего бэкапа
wal-g backup-fetch /var/lib/postgresql/17/main LATEST

# PITR
WALG_SENTINEL_USER_DATA='{"targetTime":"2026-02-25T12:00:00Z"}' \
  wal-g backup-fetch /var/lib/postgresql/17/main LATEST
```

**UI:** нет. Но есть [wal-g-exporter](https://github.com/camptocamp/wal-g-exporter) для Prometheus/Grafana.

---

### Restic

Универсальный инструмент бэкапа (не специфичен для PostgreSQL). Подходит для бэкапа файлов БД или дампов.

```bash
# Инициализация репозитория
restic -r s3:s3.amazonaws.com/my-bucket/pg init

# Сценарий: дамп + бэкап через pipe (без tmp-файла)
pg_dump -U postgres -d mydb -Fc \
  | restic -r s3:s3.amazonaws.com/my-bucket/pg backup \
    --stdin --stdin-filename mydb.dump

# Бэкап директории с файлами PG (при остановленном сервере)
restic -r /backup/restic backup /var/lib/postgresql/17/main

# Список снэпшотов
restic -r /backup/restic snapshots

# Восстановление
restic -r /backup/restic restore latest --target /restore/pg

# Политика хранения
restic -r /backup/restic forget \
  --keep-daily 7 --keep-weekly 4 --keep-monthly 6 \
  --prune
```

**UI:** [Restic Browser](https://github.com/emuell/restic-browser) — десктопное приложение для просмотра снэпшотов.

---

### Сравнение инструментов

| | pgBackRest | Barman | WAL-G | Restic |
|--|------------|--------|-------|--------|
| PostgreSQL-специфичен | ✅ | ✅ | ✅ | ❌ |
| Инкрементальный бэкап | ✅ | ✅ | ✅ (WAL) | ✅ |
| PITR | ✅ | ✅ | ✅ | ❌ |
| Облачное хранилище | ✅ | ❌ | ✅ | ✅ |
| Шифрование | ✅ | ✅ | ✅ | ✅ |
| Параллельность | ✅ | ✅ | ✅ | ✅ |
| Несколько серверов | ❌ | ✅ | ❌ | ✅ |
| Встроенный UI | ❌ | ❌ | ❌ | ❌ |

> **Рекомендация:** для продакшена — **pgBackRest** (один сервер) или **Barman** (несколько серверов). Для облака с минимальной настройкой — **WAL-G**.

---

## CDC и Kafka / Debezium

CDC (Change Data Capture) — захват изменений из WAL PostgreSQL и стриминг в Kafka. Используется для real-time синхронизации, event-driven архитектуры, аналитики.

### Архитектура

```
PostgreSQL WAL → Debezium Connector → Kafka Topic → Consumers (другой PostgreSQL, ClickHouse, ES, etc.)
```

### Настройка PostgreSQL для Debezium

```conf
# postgresql.conf
wal_level = logical
max_replication_slots = 4
max_wal_senders = 4
```

```sql
-- Создать пользователя для Debezium
CREATE USER debezium REPLICATION LOGIN PASSWORD 'password';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO debezium;

-- или через publication
CREATE PUBLICATION debezium_pub FOR ALL TABLES;
```

### Docker Compose — Kafka + Debezium

```yaml
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    depends_on: [zookeeper]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"

  kafka-connect:
    image: debezium/connect:2.7
    depends_on: [kafka]
    ports:
      - "8083:8083"
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: connect-configs
      OFFSET_STORAGE_TOPIC: connect-offsets
      STATUS_STORAGE_TOPIC: connect-status

  debezium-ui:
    image: debezium/debezium-ui:2.7
    depends_on: [kafka-connect]
    ports:
      - "8080:8080"
    environment:
      KAFKA_CONNECT_URIS: http://kafka-connect:8083
```

### Создание коннектора через REST API

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "pg-connector",
    "config": {
      "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
      "database.hostname": "postgres_host",
      "database.port": "5432",
      "database.user": "debezium",
      "database.password": "password",
      "database.dbname": "mydb",
      "database.server.name": "mydb",
      "topic.prefix": "mydb",
      "table.include.list": "public.users,public.orders",
      "plugin.name": "pgoutput",
      "publication.name": "debezium_pub",
      "slot.name": "debezium_slot",
      "snapshot.mode": "initial"
    }
  }'

# Статус коннектора
curl http://localhost:8083/connectors/pg-connector/status

# Список топиков (изменения попадают в mydb.public.users и т.д.)
kafka-topics.sh --bootstrap-server localhost:9092 --list

# Читать изменения
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic mydb.public.users \
  --from-beginning
```

### Формат сообщений Debezium

```json
{
  "op": "c",        // c=create, u=update, d=delete, r=read(snapshot)
  "before": null,   // состояние до (null для INSERT)
  "after": {        // состояние после
    "id": 1,
    "name": "Alice"
  },
  "source": {
    "db": "mydb",
    "table": "users",
    "lsn": 12345
  },
  "ts_ms": 1740477600000
}
```

### Kafka UI

Несколько вариантов:

**Debezium UI** (официальный) — управление коннекторами:
```
http://localhost:8080
```

**Kafka UI** (провия) — полный UI для Kafka:
```yaml
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8090:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_KAFKACONNECT_0_NAME: connect
      KAFKA_CLUSTERS_0_KAFKACONNECT_0_ADDRESS: http://kafka-connect:8083
```

**Redpanda Console** — альтернатива, красивый UI:
```yaml
  redpanda-console:
    image: redpandadata/console:latest
    ports:
      - "8091:8080"
    environment:
      KAFKA_BROKERS: kafka:9092
      CONNECT_ENABLED: "true"
      CONNECT_CLUSTERS_0_NAME: connect
      CONNECT_CLUSTERS_0_URL: http://kafka-connect:8083
```

### Kafka JDBC Sink — запись из Kafka обратно в PostgreSQL

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "pg-sink",
    "config": {
      "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
      "connection.url": "jdbc:postgresql://target_host:5432/targetdb",
      "connection.user": "postgres",
      "connection.password": "password",
      "topics": "mydb.public.users",
      "insert.mode": "upsert",
      "pk.mode": "record_key",
      "pk.fields": "id",
      "auto.create": "true",
      "auto.evolve": "true"
    }
  }'
```

> ⚠️ **Следи за replication slot** — `debezium_slot` накапливает WAL если consumer не читает. Мониторь: `SELECT slot_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) FROM pg_replication_slots;`

---

## UI-инструменты

### pgAdmin 4

Официальный веб-UI для PostgreSQL. Есть встроенный backup/restore.

```yaml
# Docker Compose
services:
  pgadmin:
    image: dpage/pgadmin4:latest
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin
    volumes:
      - pgadmin_data:/var/lib/pgadmin
```

Backup через UI: ПКМ на БД → Backup → выбрать формат, флаги `--no-owner`, `--no-privileges`.

### DBeaver

Десктопное приложение. Database → Tools → Backup / Restore. Под капотом вызывает `pg_dump`/`pg_restore`.

### TablePlus

Десктопное (macOS/Windows/Linux). Быстрый UI, есть базовый dump через меню.

### Adminer

Легковесный PHP-UI. Нет встроенного pg_dump, но есть экспорт таблиц в SQL/CSV.

```yaml
  adminer:
    image: adminer:latest
    ports:
      - "8080:8080"
```

---

### Бэкап

- **Проверяй бэкапы** — делай тестовое восстановление регулярно (`pg_restore --list` минимум, полное восстановление в staging идеально)
- **`-Fc` или `-Fd`** предпочтительнее `-Fp` — сжатие + параллельное восстановление
- **`-j N`** при `pg_restore` — N = количество ядер CPU, ускоряет в разы
- **Не хватает места** — используй pipe или `--compress=9`
- **Бэкап с репликой**, а не с мастером — снижает нагрузку

### Перенос между инстансами

- Всегда указывай `--no-owner --no-privileges` если переносишь на инстанс с другими ролями
- После восстановления запусти `ANALYZE` или `vacuumdb --analyze-only -d mydb`
- Последовательности (`sequences`) восстанавливаются, но текущее значение может не совпасть — проверь `SELECT last_value FROM myseq`
- `pg_upgrade` с `--link` в разы быстрее, но оригинальный кластер становится непригодным

### Логическая репликация / синхронизация

- **Replication slot** на источнике накапливает WAL пока подписчик не применит изменения — следи за `pg_replication_slots` и `pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn))` 
- Удаляй подписку (`DROP SUBSCRIPTION`) перед дропом слота
- `postgres_fdw` удобен для разовой переливки, не для постоянной синхронизации — медленнее нативной репликации
- DDL не реплицируется логически — применяй схемные изменения на обоих инстансах вручную

### Мониторинг репликации

```sql
-- Лаг репликации (на мастере)
SELECT client_addr, state, 
  pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag
FROM pg_stat_replication;

-- Статус слотов
SELECT slot_name, active,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag
FROM pg_replication_slots;
```

---

## Источники

- [pg_dump docs](https://www.postgresql.org/docs/current/app-pgdump.html)
- [pg_basebackup docs](https://www.postgresql.org/docs/current/app-pgbasebackup.html)
- [Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html)
- [pg_upgrade](https://www.postgresql.org/docs/current/pgupgrade.html)
- [pgBackRest](https://pgbackrest.org/) — рекомендуется для продакшена
- [Barman](https://pgbarman.org/) — альтернатива pgBackRest

---

## Размер баз и таблиц

```sql
-- Размер всех баз данных
SELECT datname,
       pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

-- Размер текущей БД
SELECT pg_size_pretty(pg_database_size(current_database()));

-- Размер всех таблиц (топ-20)
SELECT schemaname,
       tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total,
       pg_size_pretty(pg_relation_size(schemaname||'.'||tablename))       AS table_only,
       pg_size_pretty(pg_indexes_size(schemaname||'.'||tablename))        AS indexes
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog','information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;

-- Размер конкретной таблицы
SELECT pg_size_pretty(pg_total_relation_size('public.my_table'));

-- Размер всех индексов
SELECT indexrelname,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;

-- Размер всех схем
SELECT schema_name,
       pg_size_pretty(SUM(pg_total_relation_size(quote_ident(schema_name)||'.'||quote_ident(table_name)))) AS size
FROM information_schema.tables
WHERE schema_name NOT IN ('pg_catalog','information_schema')
GROUP BY schema_name
ORDER BY SUM(pg_total_relation_size(quote_ident(schema_name)||'.'||quote_ident(table_name))) DESC;
```

**Разница:**
- `pg_relation_size` — только данные таблицы (heap)
- `pg_indexes_size` — только индексы
- `pg_total_relation_size` — таблица + индексы + TOAST
