# PostgreSQL: Бэкап, Восстановление, Миграция, Синхронизация

## Содержание
1. [Утилиты — обзор](#утилиты--обзор)
2. [pg_dump / pg_restore](#pg_dump--pg_restore)
3. [pg_dumpall](#pg_dumpall)
4. [pg_basebackup](#pg_basebackup)
5. [PITR — Point-in-Time Recovery](#pitr--point-in-time-recovery)
6. [Перенос БД между инстансами](#перенос-бд-между-инстансами)
7. [Синхронизация и переливка данных](#синхронизация-и-переливка-данных)
8. [Рекомендации и подводные камни](#рекомендации-и-подводные-камни)

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

## Рекомендации и подводные камни

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
