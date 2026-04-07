# WebmonitorX WAF (WMX)

> Российский WAF/WAAP на базе Wallarm. Версия 4.10.
> Сертификат ФСТЭК №4899. Реестр отечественного ПО Минцифры.
> Документация: https://docs.webmonitorx.ru/

---

## Что это

WebmonitorX WAF — межсетевой экран уровня приложений (WAF/WAAP).
Защищает веб-приложения и API от атак: OWASP Top 10, боты, DDoS L7, сканирование уязвимостей.

**Технологическая база:** форк/OEM Wallarm WAF. Признаки:
- Идентичные NGINX-директивы: `wallarm_mode`, `wallarm_application`, `wallarm_fallback`
- Тот же движок анализа libproton/libdetection (проприетарный бинарь)
- Тот же Tarantool как локальная аналитическая БД на ноде
- Wallarm = американская компания; WMX = российская обёртка под регуляторику

---

## Архитектура

```
Клиент
  │
  ▼
Фильтрующая нода (on-premise, у клиента)
  │  nginx + WMX-модуль
  │  Анализ запросов ЛОКАЛЬНО (libproton)
  │  Решение о блокировке принимается здесь
  ▼
Бэкенд-приложение

  │ метрики + события атак ─────────────────────────▶ Вычислительный кластер WMX
  │                                                    (cloud, физически в России)
  │ ◀───────── обновлённые правила (ML) ───────────────
```

**Фильтрующая нода:**
- Разворачивается в инфраструктуре клиента
- Анализирует HTTP-запросы локально — трафик в облако не проксируется
- Блокирует / пропускает локально
- Отправляет в кластер: метрики, статистику атак (не тела запросов)
- Получает из кластера: обновлённые правила на основе ML

**Вычислительный кластер (RU):**
- Адреса: `my.webmonitorx.ru` (UI) и `api.webmonitorx.ru` (API)
- Хранит логи, события, отчёты
- Формирует индивидуальные правила под трафик клиента
- Сканирует защищаемые ресурсы на уязвимости (облачный сканер)

### Важно: зависимость от облака

Нода жёстко привязана к кластеру:
- Без облака нет обновлений правил и управляющей консоли
- **Fully air-gapped деплой в стандартном варианте недоступен**
- Для изолированного контура нужен enterprise-контракт с on-premise кластером

---

## Варианты развёртывания

### Linux-пакеты (APT/YUM)

Поддерживаемые ОС:
- Ubuntu 18.04 / 20.04 / 22.04
- Debian 10 / 11
- CentOS 7 / Stream 8, AlmaLinux 8, Oracle Linux 8, Rocky Linux 8

```bash
# Общий сценарий:
# 1. Создать токен в my.webmonitorx.ru
# 2. Добавить репозиторий WMX и установить пакет
# 3. Нода регистрируется по токену
# 4. Настроить nginx-директивы
# 5. Проверить: нода активна в консоли
```

### Docker

```bash
docker run -d \
  -e WMX_TOKEN=<token> \
  -e WMX_BACKEND=http://myapp:8080 \
  -e WMX_MODE=monitoring \
  -p 80:80 \
  webmonitorx/node:latest
```

Переменные: `WMX_TOKEN`, `WMX_BACKEND`, `WMX_MODE` (monitoring/blocking), `WMX_PORT`.

### Kubernetes (Helm)

```yaml
# values.yaml
wmx:
  token: "<token>"
  backend: "http://myapp-service:8080"
  mode: "monitoring"

ingress:
  enabled: true
```

```bash
helm install wmx-node webmonitorx/node -f values.yaml
```

Работает как замена Ingress Controller или sidecar рядом с существующим.

---

## Настройка NGINX

Ключевые директивы (устанавливаются в `nginx.conf`):

```nginx
wallarm_mode monitoring;      # off | monitoring | safe_blocking | block
wallarm_application 1;        # ID приложения в консоли WMX
wallarm_fallback on;          # пропускать трафик при сбое модуля
wallarm_parse_response on;    # анализировать ответы
wallarm_process_time_limit 1000;  # лимит времени анализа (мс)
```

**Важно:** модуль version-specific — должен точно совпадать с версией nginx.
При обновлении nginx нужно обновить и модуль.

---

## Интеграции

| Метод | Описание |
|---|---|
| nginx module | Основной. Встраивается в nginx как динамический модуль |
| Kubernetes Ingress | Как Ingress Controller или sidecar |
| TCP mirror | Анализ зеркала трафика (только мониторинг, без блокировки) |
| CDN | Совместимо с Cloudflare и другими CDN |
| REST API | api.webmonitorx.ru — CRUD для доменов, правил, событий, нод |
| Webhooks | Push-уведомления в SIEM / SOC |
| Email / Telegram | Алерты |

---

## Кейсы применения

- E-commerce: защита от SQLi, credential stuffing, скрапинга
- API protection: REST, GraphQL — правила на уровне эндпоинтов
- Bot mitigation: отличие краулеров от вредоносных ботов
- KII / ГИС: compliance с ФСТЭК, закрытый российский кластер
- Kubernetes: защита микросервисов на ingress уровне
- Multi-app: одна нода защищает несколько приложений (wallarm_application ID)
- MSP / MSSP: управление несколькими клиентами через API

---

## Когда WMX — не лучший выбор

| Ситуация | Альтернатива |
|---|---|
| Нужен полный air-gap | nginx + ModSecurity + OWASP CRS (open source, без облака) |
| Нет требований ФСТЭК, международный продукт | Wallarm (технически зрелее) |
| Нужно open source с полным контролем | ModSecurity / Coraza WAF |

---

## Регуляторика (Россия)

- Сертификат ФСТЭК №4899 — обязателен для ГИС, КИИ, ИСПДН
- Реестр отечественного ПО — госзакупки, льготы
- Вычислительный кластер в России — соответствие 152-ФЗ, приказам ФСТЭК
- Нет санкционных рисков (в отличие от Wallarm Inc., US)

---

## Ссылки

- Документация: https://docs.webmonitorx.ru/
- Консоль: https://my.webmonitorx.ru/
- API: https://api.webmonitorx.ru/
- Реестр ПО: https://reestr.digital.gov.ru/promo/owner/310335/794737/
- Telegram: https://t.me/WMXWAS
- Поддержка: support@wmx.pro
