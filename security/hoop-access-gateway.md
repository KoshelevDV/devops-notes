# Hoop — Access Gateway / PAM

> Прозрачный gateway для контролируемого доступа к инфраструктуре с полным аудитом сессий.

---

## Что это

[Hoop](https://github.com/hoophq/hoop) (hoophq/hoop) — self-hosted PAM (Privileged Access Management) платформа.
Разработчики подключаются к БД/серверам через hoop, не получая прямых credentials.
Все запросы логируются, записываются сессии, есть WebUI.

**Сайт:** https://hoop.dev | **Docs:** https://docs.hoop.dev
**Лицензия:** MIT (Decimals Inc.) — свободное использование, форк, модификация.

---

## Архитектура

```
Client (psql / DBeaver / hoop CLI)
    ↓  localhost proxy
hoop client
    ↓  gRPC (outbound — firewall не нужен)
hoop gateway  ←→  PostgreSQL (хранит конфиг, сессии)
    ↓
hoop agent  (рядом с целевыми ресурсами)
    ↓
PostgreSQL / MySQL / SSH / k8s / ...
```

**Три компонента (всё на Go):**
- `gateway` — control plane, REST API + gRPC, хранилище: PostgreSQL
- `agent` — лёгкий агент внутри инфраструктуры, outbound соединение до gateway
- `client` / CLI — `hoop connect`, `hoop exec`, локальный прокси

**WebApp:** React + TypeScript (ClojureScript frontend)

---

## Что даёт OSS версия (без лицензии)

| Фича | OSS |
|------|-----|
| Session recording / replay | ✅ |
| Полный audit log SQL-запросов | ✅ |
| RBAC | ✅ |
| Just-in-Time Access + Approval workflows | ✅ |
| Коннекторы: PG, MySQL, MSSQL, MongoDB, Redis, SSH, k8s exec | ✅ |
| WebUI | ✅ |
| Runbooks (параметризованные скрипты) | ✅ |
| SSO (Okta, Auth0, Google, Keycloak, Azure AD) | ✅ |
| Data masking (AI/PII маскировка) | ❌ enterprise |
| Webhooks | ❌ enterprise |

**Для localhost/127.0.0.1** — проверка хоста всегда пропускается (hardcoded в `VerifyHost`).

---

## Self-hosted — быстрый старт

Docker Compose: `deploy/docker-compose/docker-compose.yml`

Поднимает: gateway + agent + postgres + Microsoft Presidio (PII analyzer/anonymizer).

```bash
git clone https://github.com/hoophq/hoop
cd hoop/deploy/docker-compose
cp .env-sample .env
# заполнить .env (AUTH_METHOD, POSTGRES_DB_URI и т.д.)
docker compose up -d
```

**Порты gateway:**
- `8009` — HTTP API / WebUI
- `8010` — gRPC (agent)
- `15432` — PostgreSQL proxy
- `12222` — SSH proxy
- `13389` — RDP proxy

**Helm (k8s):**
```bash
helm repo add hoophq https://hoophq.github.io/helm-charts
helm install hoop hoophq/hoop
```

---

## Лицензионный механизм

RSA-подписанный токен (PSS). Публичный ключ захардкожен в бинаре.
Типы: `oss` (default, без токена) и `enterprise`.
Патч возможен: заменить pubkey в `common/license/license.go` при сборке из исходников + сгенерировать свой enterprise-токен.
Для dev-использования не нужно — OSS достаточно.

---

## Конкуренты

| Проект | Лицензия | Особенности |
|--------|----------|-------------|
| [Teleport](https://github.com/gravitational/teleport) | Apache 2.0 / Enterprise | Более зрелый, сложнее |
| [HashiCorp Boundary](https://github.com/hashicorp/boundary) | MPL 2.0 | Часть Vault экосистемы |
| [StrongDM](https://strongdm.com) | Проприетарный | SaaS-only |

---

## Use case — прозрачный gateway для dev

Задача: видеть что делают разработчики в СУБД, не раздавая прямые credentials.

```bash
# разработчик:
hoop connect postgres-dev
# → локальный прокси на localhost:5432
# → все запросы пишутся в audit log + session replay в WebUI
# psql / DBeaver работают без изменений
```
