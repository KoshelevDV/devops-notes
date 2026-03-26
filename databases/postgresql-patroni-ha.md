# HA PostgreSQL: etcd + Patroni + HAProxy + keepalived

**Источник:** https://kodikapusta.ru/news/909-postgresql-s-patroni  
**Автор:** Шон Томас (серия из 3 частей)  
**Дата заметки:** 2026-03-26

---

## Архитектура полноценного кластера

```
Клиенты
   │
[keepalived VIP]       ← виртуальный IP, failover между HAProxy нодами
   │
[HAProxy #1] [HAProxy #2]   ← балансировка, мониторинг через Patroni REST API
   │
[Patroni + PG Primary]
[Patroni + PG Replica 1]
[Patroni + PG Replica 2]
   │
[etcd cluster]         ← distributed state store (DCS)
```

---

## Части серии

1. **Часть 1** — etcd как distributed consensus store
2. **Часть 2** — установка и настройка Patroni + PostgreSQL
3. **Часть 3** — HAProxy как слой маршрутизации

---

## Как HAProxy работает с Patroni

HAProxy использует **REST API Patroni** для health checks:
- `HTTP 200 OK` → нода является primary → направлять трафик сюда
- Можно настроить отдельный backend для реплик (с проверкой lag)
- При failover Patroni меняет роли → HAProxy автоматически перестраивает маршрутизацию

```haproxy
backend postgres_primary
    option httpchk GET /primary
    http-check expect status 200
    server pg1 10.0.0.1:5432 check port 8008
    server pg2 10.0.0.2:5432 check port 8008
    server pg3 10.0.0.3:5432 check port 8008
```

Patroni слушает на порту `8008` (по умолчанию) и отвечает:
- `/primary` → 200 если нода leader, 503 иначе
- `/replica` → 200 если нода replica
- `/replica?lag=10MB` → 200 если replica и отставание < 10MB

---

## Важно: HAProxy — SPOF без keepalived

Если HAProxy один — он сам становится точкой отказа.

**Решение:** два HAProxy + keepalived с виртуальным IP:
- keepalived держит VIP на активной ноде HAProxy
- При падении HAProxy#1 — VIP переходит на HAProxy#2
- Клиенты всегда подключаются к VIP, не замечают failover

```
keepalived конфиг (мастер):
  virtual_ipaddress: 10.0.0.100/24
  priority: 100

keepalived конфиг (бэкап):
  virtual_ipaddress: 10.0.0.100/24
  priority: 90
```

---

## Итоговая отказоустойчивость

| Компонент | Что происходит при падении |
|-----------|--------------------------|
| PostgreSQL primary | Patroni проводит failover, выбирает новый primary из реплик |
| HAProxy нода | keepalived переключает VIP на вторую HAProxy ноду |
| etcd нода | Кластер etcd продолжает работу при наличии кворума (N/2+1) |
| Patroni нода | Остальные ноды продолжают работу, возможен re-election |

---

## Минимальное число нод

- **etcd**: 3 ноды (кворум при 1 отказе) или 5 (при 2 отказах)
- **Patroni+PG**: 3 ноды (1 primary + 2 replica)
- **HAProxy**: 2 ноды + keepalived
