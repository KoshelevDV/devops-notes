# Nginx: наследование proxy_set_header и приоритеты директив

> **Дата:** 2026-02-26  
> **Теги:** `nginx` `proxy` `headers` `inheritance` `конфиги`

---

## 🎯 Проблема

В nginx цепочка конфигов (http → server → location) наследует директивы **не так, как кажется**.
Самая частая жертва — `proxy_set_header Host`.

Реальный кейс: в глобальном конфиге задан `proxy_set_header Host $host;`, но Next.js в upstream
получает `Host: lb-to-docker` (имя upstream-блока). Причина — в одном из `location` есть
`proxy_set_header Connection "";` без явного `Host`.

---

## ✅ Главное правило наследования

**Директивы `proxy_set_header` НЕ мёржатся между уровнями.**  
Если в дочернем контексте (location) есть хотя бы ОДИН `proxy_set_header` —
весь родительский блок `proxy_set_header` (http/server) полностью игнорируется.

```
http {
    proxy_set_header Host $host;           # ← задан глобально
    proxy_set_header X-Forwarded-For ...;

    server {
        location /api {
            proxy_pass http://backend;
            proxy_set_header Connection "";  # ← есть хоть один

            # Host и X-Forwarded-For из http{} уже НЕ применяются!
            # nginx ставит Host = $proxy_host = имя upstream = "backend"
        }
    }
}
```

То же самое работает для: `proxy_hide_header`, `proxy_pass_header`, `add_header`,
`fastcgi_param`, `uwsgi_param`.

---

## 🔍 Как диагностировать

### 1. Посмотреть полный итоговый конфиг со всеми include
```bash
nginx -T 2>/dev/null | grep -A10 -B5 "location /нужный-path"
```

### 2. Найти все proxy_set_header в locations
```bash
grep -rn "proxy_set_header" /etc/nginx/
# или в нестандартном расположении:
grep -rn "proxy_set_header" /var/www/project/nginx/locations/
```

### 3. Временный дебаг-эндпоинт — что реально видит upstream
```typescript
// pages/api/debug-headers.ts (Next.js, удалить после отладки)
export default function handler(req, res) {
  res.status(200).json({
    host: req.headers.host,
    'x-forwarded-host': req.headers['x-forwarded-host'],
    'x-forwarded-for': req.headers['x-forwarded-for'],
    all: req.headers,
  });
}
```

```bash
curl https://example.com/api/debug-headers
```

### 4. Стучаться напрямую на каждое звено цепи с нужным Host
```bash
# Проверить что upstream реально получает
curl -H "Host: www.example.com" http://<ip-сервера>/robots.txt | grep Sitemap
```

---

## ✅ Решение

### Вариант 1 — явно добавить Host в каждый location
```nginx
location /api {
    proxy_pass http://backend;
    proxy_set_header Host $host;              # ← всегда явно
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header Connection "";
}
```

### Вариант 2 — вынести общие заголовки в отдельный include
```nginx
# nginx/proxy-headers.conf
proxy_set_header Host $host;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Real-IP $remote_addr;

# в каждом location:
location /api {
    include nginx/proxy-headers.conf;
    proxy_set_header Connection "";   # специфичный для этого location
    proxy_pass http://backend;
}
```

### Вариант 3 — не дублировать, вынести всё на уровень server{}
Если большинство locations проксируют на один upstream, лучше один раз задать всё в `server {}`:

```nginx
server {
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Connection "";   # общий для всех

    location /api { proxy_pass http://backend; }
    location /app { proxy_pass http://app; }
    # locations без своих proxy_set_header наследуют всё из server{}
}
```

---

## 💡 Подводные камни

- **`$host` vs `$http_host`** — разные переменные:
  - `$http_host` — буквально заголовок Host из запроса (с портом, если был: `example.com:8080`)
  - `$host` — имя сервера: из Host-заголовка без порта, lowercase; если нет → server_name
  - Для `proxy_set_header Host` обычно нужен `$host`

- **`$proxy_host`** — имя и порт из директивы `proxy_pass`. Это дефолтное значение Host
  если proxy_set_header Host не задан явно. При upstream-блоке = его имя (`lb-to-docker`).

- **`proxy_set_header Connection ""`** — нужен для HTTP keepalive к upstream,
  часто добавляется в locations отдельно — и именно он ломает наследование Host.

- **Правило применяется ко всем "блочным" директивам:**
  `add_header`, `proxy_set_header`, `fastcgi_param`, `uwsgi_param`, `scgi_param`

- **Разные конфиги на нодах кластера** — деплой должен быть идентичным.
  Ручные правки "для теста" оставленные в проде — частый источник таких багов.

---

## 📎 Документация

- [Nginx: proxy_set_header — официальная дока](https://nginx.org/ru/docs/http/ngx_http_proxy_module.html#proxy_set_header)
- [Nginx: как работает наследование директив](https://nginx.org/ru/docs/directives-merge.html)

---

## 🔗 Источники

- Разбор реального бага: robots.txt возвращал технический upstream-хост вместо публичного домена
