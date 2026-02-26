# Nginx: визуализация и разбор спагетти конфигов

> **Дата:** 2026-02-26  
> **Теги:** `nginx` `конфиги` `отладка` `include` `инструменты`

---

## 🎯 Проблема

Nginx конфиг разбит на десятки файлов с include в include — непонятно что откуда берётся,
какой location срабатывает, и где вообще нужная директива.

---

## ✅ Инструменты

### 1. `nginx -T` — склеить всё в один файл (самое быстрое)

```bash
nginx -T 2>/dev/null > /tmp/nginx-full.conf

# Теперь grep без боли
grep -n "location\|proxy_pass\|upstream\|server_name" /tmp/nginx-full.conf

# Найти откуда пришла конкретная директива
grep -n "proxy_set_header Host" /tmp/nginx-full.conf
```

### 2. Дерево include-ов (bash-скрипт)

```bash
#!/bin/bash
show_includes() {
    local file="$1"
    local indent="$2"
    echo "${indent}${file}"
    grep -hE "^\s*include\s+" "$file" 2>/dev/null | grep -v '#' | \
    awk '{print $NF}' | tr -d ';' | while read -r pattern; do
        for f in $pattern; do
            [ -f "$f" ] && show_includes "$f" "${indent}  └─ "
        done
    done
}
show_includes /etc/nginx/nginx.conf ""
```

Пример вывода:
```
/etc/nginx/nginx.conf
  └─ /etc/nginx/conf.d/default.conf
       └─ /etc/nginx/locations/front.conf
       └─ /etc/nginx/locations/api.conf
            └─ /etc/nginx/locations/front-template-static.tpl
```

### 3. `crossplane` — парсит конфиг в JSON

```bash
pip install crossplane
crossplane parse /etc/nginx/nginx.conf | python3 -m json.tool > /tmp/nginx.json

# Все proxy_pass в конфиге
crossplane parse /etc/nginx/nginx.conf | \
  python3 -c "
import json,sys
d=json.load(sys.stdin)
def find(obj, key):
    if isinstance(obj, list):
        for i in obj: find(i, key)
    elif isinstance(obj, dict):
        if obj.get('directive') == key:
            print(obj.get('args'), obj.get('file',''), 'line', obj.get('line',''))
        for v in obj.values(): find(v, key)
find(d, 'proxy_pass')
"

# Или через jq:
cat /tmp/nginx.json | jq '.. | objects | select(.directive=="proxy_pass") | {args,file,line}'
```

### 4. `gixy` — статический анализ (+ находит уязвимости)

```bash
pip install gixy
gixy /etc/nginx/nginx.conf
```

### 5. Какой location сработает для запроса

```bash
# Показывает какой location nginx выберет для URI
nginx -T | grep -A5 "location"

# Или через тестовый запрос с debug-логом
error_log /tmp/nginx-debug.log debug;  # временно в конфиге
tail -f /tmp/nginx-debug.log | grep "using configuration"
```

### 6. Онлайн-инструменты (если конфиг не секретный)

- https://nginx-playground.com — интерактивный sandbox
- https://www.digitalocean.com/community/tools/nginx — генератор + визуализатор
- https://jsoncrack.com — вставить JSON от crossplane

---

## 💡 Подводные камни

- `nginx -T` показывает конфиг **как он есть на диске**, не как nginx его видит в памяти.
  После `nginx -s reload` старый конфиг в памяти — проверяй через `nginx -t` перед reload.

- include с glob (`include locations/*.conf`) — порядок файлов алфавитный.
  Если порядок важен — нумеруй файлы: `10-front.conf`, `20-api.conf`.

- Директива `location` матчится по правилам приоритета (не по порядку в файле):
  `=` > `^~` > regex (`~`, `~*`) > prefix. Это отдельная тема — см. заметку по location-приоритетам.

---

## 📎 Документация

- [crossplane на GitHub](https://github.com/nginxinc/crossplane)
- [gixy — nginx security scanner](https://github.com/yandex/gixy)
- [Nginx: debugging log](https://nginx.org/en/docs/debugging_log.html)
