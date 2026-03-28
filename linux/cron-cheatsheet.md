# Cron — шпаргалка

## Посмотреть кроны всех пользователей

```bash
# Fedora/RHEL/Debian/Ubuntu — список crontab всех юзеров
for user in $(cut -f1 -d: /etc/passwd); do
  crontab -u "$user" -l 2>/dev/null && echo "^^^ $user"
done

# Короче — только те у кого есть crontab
for user in $(cut -f1 -d: /etc/passwd); do
  crontab -u "$user" -l 2>/dev/null | grep -v '^#' | grep -v '^$' \
    && echo "--- user: $user ---"
done

# Прямой просмотр файлов (если root)
ls -la /var/spool/cron/crontabs/   # Debian/Ubuntu
ls -la /var/spool/cron/            # RHEL/Fedora/CentOS

cat /var/spool/cron/crontabs/*     # все crontab (Debian/Ubuntu)
cat /var/spool/cron/*              # все crontab (RHEL/Fedora)
```

## Системные cron-задачи

```bash
# Файлы в /etc/cron.d/ (с указанием пользователя в строке)
cat /etc/cron.d/*

# Директории с периодическими скриптами
ls /etc/cron.daily/
ls /etc/cron.hourly/
ls /etc/cron.weekly/
ls /etc/cron.monthly/

# Основной crontab системы
cat /etc/crontab
```

## На Fedora/RHEL — systemd timers (заменяют cron)

```bash
# Все таймеры (включая неактивные)
systemctl list-timers --all

# User-level таймеры конкретного пользователя
systemctl --user list-timers --all

# Посмотреть все user-timers всех пользователей (root)
loginctl list-users | awk 'NR>1 && NF>=2 {print $1, $2}' | while read uid user; do
  echo "=== $user ==="; systemctl -M "$user@" --user list-timers --all 2>/dev/null
done
```

---

## Посмотреть кроны в БД PostgreSQL

### Расширение pg_cron (если установлено)

```sql
-- Список всех заданий
SELECT jobid, jobname, schedule, command, nodename, active
FROM cron.job
ORDER BY jobid;

-- История последних запусков
SELECT jobid, job_pid, database, username, command,
       status, return_message, start_time, end_time
FROM cron.job_run_details
ORDER BY start_time DESC
LIMIT 50;

-- Только активные задания
SELECT * FROM cron.job WHERE active = true;

-- Задания с последним статусом
SELECT j.jobname, j.schedule, j.command, r.status, r.start_time
FROM cron.job j
LEFT JOIN LATERAL (
    SELECT status, start_time
    FROM cron.job_run_details
    WHERE jobid = j.jobid
    ORDER BY start_time DESC
    LIMIT 1
) r ON true
ORDER BY j.jobid;
```

### Проверить установлено ли pg_cron

```sql
SELECT * FROM pg_extension WHERE extname = 'pg_cron';

-- Или через psql
\dx pg_cron
```

### Установить pg_cron (если нужно)

```bash
# Fedora/RHEL
sudo dnf install pg_cron_17   # или нужная версия PG

# В postgresql.conf добавить:
shared_preload_libraries = 'pg_cron'
cron.database_name = 'postgres'

# После рестарта PG:
CREATE EXTENSION pg_cron;
```

### Если pg_cron нет — кроны могут быть в application-таблицах

```sql
-- Ищем таблицы с "job", "task", "schedule", "cron" в названии
SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_name ILIKE '%job%'
   OR table_name ILIKE '%task%'
   OR table_name ILIKE '%schedul%'
   OR table_name ILIKE '%cron%'
ORDER BY table_schema, table_name;
```

---

## Быстрый аудит — всё сразу

```bash
#!/bin/bash
echo "=== /etc/crontab ==="
cat /etc/crontab 2>/dev/null

echo -e "\n=== /etc/cron.d/ ==="
cat /etc/cron.d/* 2>/dev/null

echo -e "\n=== User crontabs ==="
for user in $(cut -f1 -d: /etc/passwd); do
  crontab -u "$user" -l 2>/dev/null | grep -v '^#' | grep -v '^$' \
    && echo "--- $user ---"
done

echo -e "\n=== systemd timers ==="
systemctl list-timers --all --no-pager
```
