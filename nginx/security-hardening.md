# Nginx: безопасность конфигов и приложений за ним

> **Дата:** 2026-02-26  
> **Теги:** `nginx` `security` `hardening` `headers` `tls` `waf`

---

## 🎯 Цель

Закрыть nginx так, чтобы злоумышленники не нашли точку входа — ни через сам nginx,
ни через приложение за ним.

---

## 1. Базовый харденинг nginx

### Скрыть версию и токен сервера

```nginx
http {
    server_tokens off;                    # убрать версию из заголовков и страниц ошибок
    # more_clear_headers Server;          # если установлен модуль headers-more — убрать вообще
}
```

Проверить:
```bash
curl -I https://example.com | grep -i server
# Должно быть: Server: nginx (без версии) или вообще ничего
```

### Закрыть лишние методы

```nginx
location / {
    limit_except GET POST HEAD {
        deny all;
    }
}
```

### Запретить доступ к скрытым файлам

```nginx
location ~ /\. {
    deny all;
    access_log off;
    log_not_found off;
}
# Защищает .git, .env, .htpasswd и т.д.
```

### Ограничить размер запроса

```nginx
client_max_body_size 10m;      # под нужды приложения
client_body_timeout 10s;
client_header_timeout 10s;
```

---

## 2. TLS — правильная настройка

```nginx
server {
    listen 443 ssl;
    http2 on;

    ssl_certificate     /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    # Только современные протоколы
    ssl_protocols TLSv1.2 TLSv1.3;

    # Безопасные шифры (Mozilla Modern)
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers off;   # для TLS 1.3 off — клиент выбирает

    # Кэш сессий
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;         # отключить — PFS

    # OCSP stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 1.1.1.1 8.8.8.8 valid=300s;
}

# Редирект HTTP → HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

Проверить:
```bash
testssl.sh example.com              # https://testssl.sh/
# или онлайн: https://www.ssllabs.com/ssltest/
```

---

## 3. Security Headers

```nginx
# В server{} или http{}
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;

# Content-Security-Policy — под конкретное приложение, не универсально
# add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'" always;
```

⚠️ **Важно:** `add_header` тоже не наследуется если в location есть свой `add_header` — тот же баг что с `proxy_set_header`. Использовать `always` и выносить в include.

Проверить:
```bash
curl -I https://example.com | grep -iE "strict|frame|content-type|xss|referrer"
# или онлайн: https://securityheaders.com
```

---

## 4. Rate limiting — защита от брутфорса и DDoS

```nginx
http {
    # Зоны ограничений
    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    server {
        # Общий лимит
        limit_req zone=general burst=20 nodelay;
        limit_conn conn_limit 20;

        # Жёсткий лимит для auth-эндпоинтов
        location /api/auth/ {
            limit_req zone=login burst=5 nodelay;
            proxy_pass http://backend;
        }
    }
}
```

---

## 5. Защита от утечки внутренней инфраструктуры

```nginx
# Не светить внутренние адреса в заголовках ответа
proxy_hide_header X-Powered-By;
proxy_hide_header X-Application-Context;
proxy_hide_header Server;           # если upstream тоже шлёт

# Не пробрасывать технические заголовки клиенту
proxy_hide_header X-Backend-Server;
proxy_hide_header X-Upstream-Addr;
```

```bash
# Проверить что не утекает:
curl -I https://example.com | grep -iE "x-powered|x-backend|x-upstream|x-application"
```

---

## 6. Закрыть доступ к чувствительным путям

```nginx
# .env, бэкап-файлы, редакторные артефакты
location ~* \.(env|bak|sql|swp|old|orig|log)$ {
    deny all;
}

# Управлялки и технические пути — только с доверенных IP
location /admin/ {
    allow 10.0.0.0/8;
    allow 192.168.0.0/16;
    deny all;
    proxy_pass http://backend;
}

# Заблокировать типичные сканеры
location ~* (wp-login|phpMyAdmin|\.php$|actuator|\.git) {
    deny all;
    access_log off;
}
```

---

## 7. Проверка конфига на уязвимости — `gixy`

```bash
pip install gixy
gixy /etc/nginx/nginx.conf
```

Что ищет:
- **SSRF** через `proxy_pass` с переменными
- **Path traversal** через alias + location баг
- **Host header injection**
- **Небезопасный add_header** (без `always`)
- **HTTP splitting** через незащищённые переменные в заголовках

---

## 8. Мониторинг и логи

```nginx
# Формат лога с нужными полями
log_format security '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    '$request_time $upstream_response_time';

access_log /var/log/nginx/access.log security;

# Отдельный лог для ошибок 4xx/5xx
map $status $loggable {
    ~^[23] 0;
    default 1;
}
access_log /var/log/nginx/errors.log security if=$loggable;
```

Алёрты (fail2ban или аналог):
```bash
# Пример: бан IP при >100 запросах в минуту с 4xx
# /etc/fail2ban/filter.d/nginx-4xx.conf
[Definition]
failregex = ^<HOST> .* "(GET|POST|HEAD).*" (4\d\d) .*$
ignoreregex =

# /etc/fail2ban/jail.local
[nginx-4xx]
enabled  = true
filter   = nginx-4xx
logpath  = /var/log/nginx/access.log
maxretry = 100
findtime = 60
bantime  = 3600
```

---

## 9. Чеклист перед выходом в прод

```
[ ] server_tokens off
[ ] Только TLS 1.2/1.3, слабые шифры отключены
[ ] Security headers все выставлены (HSTS, X-Frame, CSP и т.д.)
[ ] .env, .git, бэкапы закрыты
[ ] Rate limiting на auth-эндпоинтах
[ ] /admin закрыт по IP или mTLS
[ ] proxy_hide_header убрал X-Powered-By и внутренние заголовки
[ ] gixy прогнал — нет предупреждений
[ ] ssllabs.com — рейтинг A или A+
[ ] securityheaders.com — рейтинг A или A+
[ ] Логи пишутся, алёрты настроены
```

---

## 📎 Документация

- [Mozilla SSL Config Generator](https://ssl-config.mozilla.org/) — генератор правильного TLS
- [OWASP Nginx Security](https://owasp.org/www-project-secure-headers/)
- [gixy — nginx security scanner](https://github.com/yandex/gixy)

---

## 🔗 Источники

- [Hardening nginx (HackerNews threads)](https://news.ycombinator.com)
- [testssl.sh](https://testssl.sh/)
- [securityheaders.com](https://securityheaders.com)
- [ssllabs.com/ssltest](https://www.ssllabs.com/ssltest/)
