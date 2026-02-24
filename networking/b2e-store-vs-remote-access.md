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
| `store-private` | Приватный WiFi → корп. сеть | RFC1918 IP, внутренний домен, высокое доверие |
| `store-public` | Публичный WiFi магазина | Публичный IP магазина, среднее доверие |
| `remote` | Вне магазина | Случайный IP, минимальное доверие |

**Ключевой момент:**
- **Приватный WiFi** → трафик идёт через корп. сеть → на B2E попадает с **RFC1918 IP** (10.x / 192.168.x). Это делает `store-private` легко различимым и позволяет добавить deny для публичных IP на внутреннем эндпоинте.
- **Публичный WiFi** → NAT через публичный IP магазина → source IP = IP магазина. Определяется через IP ACL.
- Оба WiFi NATятся через один публичный IP магазина (если бы приватный тоже шёл наружу) — но поскольку приватный идёт во внутреннюю сеть, это **не проблема**.

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

    # Двойная верификация: внутренний домен + приватный IP
    # Трафик с приватного WiFi идёт через корп. сеть → source IP = RFC1918
    acl is_corp_ip src 10.0.0.0/8 192.168.0.0/16 172.16.0.0/12
    http-request deny unless is_corp_ip  # публичный IP на внутренний эндпоинт = отказать

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

## 🔍 Получение реального IP клиента

HAProxy должен видеть реальный IP, а не IP Mikrotik. Разбор по случаям:

| Путь | Что видит HAProxy | Причина |
|------|------------------|---------|
| Приватный WiFi | RFC1918 IP клиента ✅ | Mikrotik роутит без NAT |
| Публичный WiFi | Публичный IP магазина ✅ | Нужный IP для ACL — всё ок |
| Удалёнка | Реальный IP юзера ✅ | Прямое подключение |

**Для приватного WiFi — проверить Masquerade на Mikrotik:**
```bash
# Mikrotik CLI
/ip firewall nat print
```
Если есть `action=masquerade` на маршруте internal→internal — HAProxy будет видеть IP Mikrotik вместо клиента. Добавить исключение:
```
/ip firewall nat add chain=srcnat \
    src-address=10.0.0.0/8 dst-address=10.0.0.0/8 \
    action=accept place-before=0
```

**HAProxy — правильная обработка X-Forwarded-For:**
```haproxy
frontend b2e_public
    option forwardfor
    # Доверять XFF только от известных внутренних прокси
    http-request set-header X-Real-IP %[req.hdr(X-Forwarded-For,-1)] \
        if { src 10.0.0.0/8 192.168.0.0/16 }
    http-request set-header X-Real-IP %[src] \
        if !{ req.hdr(X-Forwarded-For) -m found }
```

**Если перед HAProxy ещё один балансировщик — PROXY Protocol (лучший вариант):**
```haproxy
frontend b2e_public
    bind *:443 ssl crt /etc/ssl/certs/b2e.pem accept-proxy
```

---

## 💡 Подводные камни

- **IP-список надо обновлять** при открытии/переезде магазина. Желательно автоматизировать через Ansible/скрипт + reload HAProxy (`haproxy -sf $(cat /run/haproxy.pid)`)
- **Приватный WiFi → RFC1918**: трафик приватного WiFi приходит с корп. IP, а не с публичным IP магазина. Это **хорошо** — не нужно включать его в IP ACL, он сам детектируется по RFC1918 диапазону на внутреннем фронтенде.
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
