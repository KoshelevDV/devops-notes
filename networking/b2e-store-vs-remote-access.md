# Разделение типов доступа к B2E: магазин vs удалёнка

> **Дата:** 2026-02-24  
> **Теги:** `networking` `b2e` `haproxy` `nginx` `mikrotik` `api-gateway` `access-control`

---

## 🎯 Проблема

Ритейл-компания, сеть магазинов. В каждом магазине:
- **Приватный WiFi** — доступ к корп. сети + интернет
- **Публичный WiFi** — только интернет, без корп. сети
- Оба WiFi только для сотрудников
- У каждого магазина один статический белый IP

Есть B2E-приложение (API Gateway) — торчит наружу, доступен и из корп. сети.

**Нужно разделить три типа подключения:**

| Тип | Откуда | Характеристика |
|-----|--------|----------------|
| `store-private` | Приватный WiFi магазина | Корп. сеть, высокое доверие |
| `store-public` | Публичный WiFi магазина | Публичный IP магазина, среднее доверие |
| `remote` | Вне магазина | Случайный IP, минимальное доверие |

**Сложность:** оба WiFi в магазине NATятся через один публичный IP. IP-адрес один для обоих — IP-анализом одного не отличить от другого.

**Стек:** Mikrotik → HAProxy → Nginx Ingress → API Gateway

---

## ✅ Решение: двухуровневая маркировка

### Уровень 1 — Внутренний домен (определяет `store-private`)

На приватном WiFi Mikrotik резолвит `b2e-internal.corp` во **внутренний** IP HAProxy/Nginx.  
Публичный WiFi и удалёнка этот домен не резолвят → клиент падает на публичный домен.

**Mikrotik — статическая DNS-запись на VLAN приватного WiFi:**
```
/ip dns static
add name=b2e-internal.corp address=<внутренний IP HAProxy> ttl=30s
```

> Только устройства на приватном VLAN получат эту запись. Публичный VLAN — нет.

**B2E клиент — логика fallback:**
```javascript
async function getApiBase() {
  try {
    const ctrl = new AbortController();
    setTimeout(() => ctrl.abort(), 2000); // 2 сек таймаут
    await fetch('https://b2e-internal.corp/health', { signal: ctrl.signal });
    return 'https://b2e-internal.corp'; // приватная сеть
  } catch {
    return 'https://b2e.company.com'; // публичный домен
  }
}
```

---

### Уровень 2 — IP ACL (определяет `store-public` vs `remote`)

Для трафика, пришедшего на публичный домен, HAProxy сверяет source IP со списком публичных IP магазинов.

**Файл `/etc/haproxy/store-ips.lst`:**
```
203.0.113.10    # Магазин Москва-1
203.0.113.11    # Магазин Москва-2
198.51.100.5    # Магазин СПб-1
# ...
```

---

### Уровень 3 — Простановка заголовка X-Access-Type

**HAProxy config:**
```haproxy
frontend b2e_public
    bind *:443 ssl crt /etc/ssl/certs/b2e.pem

    # ВАЖНО: всегда удалять входящий заголовок — клиент не должен его подделать
    http-request del-header X-Access-Type

    acl is_store_ip src -f /etc/haproxy/store-ips.lst
    http-request set-header X-Access-Type store-public if is_store_ip
    http-request set-header X-Access-Type remote         if !is_store_ip

    default_backend nginx_ingress

frontend b2e_internal
    bind <внутренний IP>:443 ssl crt /etc/ssl/certs/b2e-internal.pem

    http-request del-header X-Access-Type
    http-request set-header X-Access-Type store-private

    default_backend nginx_ingress
```

**Nginx Ingress — пробросить заголовок в API Gateway:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: b2e-ingress
  annotations:
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_pass_header X-Access-Type;
```

**API Gateway — читает `X-Access-Type` и применяет политику:**
```
store-private → полный доступ к корп. функциям
store-public  → ограниченный доступ (без чувствительных операций)
remote        → минимальный доступ + строгий rate limit
```

---

## 🍪 Header vs Cookie — сравнение

| | **X-Access-Type Header** | **Cookie** |
|--|--------------------------|-----------|
| Кто выставляет | Инфраструктура (HAProxy) | Сервер в ответе |
| Подделка клиентом | Невозможна (HAProxy удаляет входящий) | Технически можно (если не HttpOnly) |
| Сессионность | Каждый запрос маркируется заново | Персистентен между запросами |
| Прозрачность | Видна в инструментах разработчика | Скрыта (HttpOnly) |
| **Вывод** | ✅ Основной механизм безопасности | ⚠️ Только для UX/персонализации |

**Рекомендация:** заголовок — для контроля доступа, cookie — только если нужно запомнить тип подключения в сессии для UI (например, показывать разные интерфейсы).

---

## 💡 Подводные камни

- **IP-список надо обновлять** при открытии/переезде магазина. Желательно автоматизировать через Ansible/скрипт + reload HAProxy (`haproxy -sf $(cat /run/haproxy.pid)`)
- **Hairpin NAT**: если публичный WiFi тоже ходит на публичный IP (что скорее всего) — он попадёт в `store-public`, а не в `store-private`. Это правильно и ожидаемо.
- **Таймаут fallback** в клиенте: 2 сек — компромисс. Слишком мал → ложные срабатывания на медленном WiFi. Слишком велик → долгий старт. Тестировать на реальной сети.
- **DNS TTL**: ставь маленький TTL (30–60s) для `b2e-internal.corp` — чтобы при смене VLAN не цеплялся старый кэш.
- **Внутренний TLS**: для `b2e-internal.corp` нужен сертификат. Варианты: корпоративный CA, wildcard, или self-signed с пином в клиенте.
- **X-Forwarded-For**: если HAProxy за другим балансировщиком — убедись, что смотришь на правильный IP (не на IP промежуточного прокси). Использовать `src` в HAProxy ACL — это реальный IP TCP-соединения.

---

## 📎 Документация

- [HAProxy ACL — официальная дока](https://cbonte.github.io/haproxy-dconv/2.8/configuration.html#7)
- [HAProxy http-request set-header](https://cbonte.github.io/haproxy-dconv/2.8/configuration.html#http-request%20set-header)
- [Mikrotik DNS Static](https://help.mikrotik.com/docs/display/ROS/DNS)
- [Nginx Ingress — configuration-snippet](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#configuration-snippet)

---

## 🔗 Источники

- [Обсуждение split-DNS для корп. сетей — Reddit r/networking](https://www.reddit.com/r/networking/comments/split_dns_corporate/)
- Simon Willison — "Lethal trifecta" (доверие к источнику запроса в AI-агентах, но принцип применим шире)
