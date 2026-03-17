# MTProto Proxy с Fake TLS

> Self-hosted прокси для Telegram. Трафик маскируется под HTTPS через Fake TLS — практически неотличим от обычного сайта.

---

## Стек

- **Image:** `mtprotoproxy` (alexbers/mtprotoproxy, Python) — собирается локально из Alpine
- **Transport:** Fake TLS (секрет с префиксом `ee` + hex домена)
- **Маскировка:** трафик притворяется HTTPS-запросами к `ya.ru`

---

## Деплой на 64.188.63.31 (Timeweb Cloud)

### Структура
```
/opt/mtproto/
├── Dockerfile
└── config.py
```

### Dockerfile
```dockerfile
FROM alpine:3.15
RUN apk add --no-cache python3 py3-pip git && \
    git clone https://github.com/alexbers/mtprotoproxy.git /app
WORKDIR /app
CMD ["python3", "mtprotoproxy.py"]
```

### config.py
```python
PORT = 443
USERS = {
    "tg": "ee79612e72752ddd281f54079f79cbb8"
}
TLS_DOMAIN = "ya.ru"
```

### Запуск
```bash
cd /opt/mtproto
sudo docker build -t mtproto-proxy .
sudo docker run -d \
  --name mtproto-proxy \
  --restart unless-stopped \
  -p 8443:443 \
  -v /opt/mtproto/config.py:/app/config.py \
  mtproto-proxy
```

### UFW
```bash
sudo ufw allow 8443/tcp comment 'MTProto Fake TLS'
```

---

## Подключение

**Ссылка:**
```
tg://proxy?server=64.188.63.31&port=8443&secret=ee79612e72752ddd281f54079f79cbb8
```

**Вручную в Telegram:**
- Мобильный: Настройки → Данные и память → Настройки прокси → Добавить прокси → MTProto
- Десктоп: Настройки → Продвинутые настройки → Тип соединения → Использовать собственный прокси → MTProto

| Поле   | Значение                           |
|--------|------------------------------------|
| Сервер | 64.188.63.31                       |
| Порт   | 8443                               |
| Секрет | ee79612e72752ddd281f54079f79cbb8   |

---

## Управление

```bash
# SSH на сервер
ssh -i <key> -p 22022 user@64.188.63.31

# Статус
sudo docker ps | grep mtproto
sudo docker logs mtproto-proxy --tail 20

# Рестарт
sudo docker restart mtproto-proxy

# Остановить
sudo docker stop mtproto-proxy
```

---

## Генерация нового секрета

```bash
FAKE_DOMAIN="ya.ru"
DOMAIN_HEX=$(echo -n $FAKE_DOMAIN | xxd -ps | tr -d '\n')
RANDOM_HEX=$(openssl rand -hex 15 | cut -c1-$((30 - ${#DOMAIN_HEX})))
SECRET="ee${DOMAIN_HEX}${RANDOM_HEX}"
echo "tg://proxy?server=64.188.63.31&port=8443&secret=$SECRET"
```

---

## Проверка работы

```bash
# Порт открыт
curl -v --max-time 5 https://64.188.63.31:8443 -k 2>&1 | grep -E "TLS|Connected|HTTP"
# Ожидаемо: TLS handshake OK, HTTP/2 406 (прокси ждёт MTProto трафик)
```

---

## Заметки

- 443 порт уже занят XRay/TLS → прокси поднят на **8443**
- DockerHub rate limit — образ собирается локально из Alpine (нет зависимости от registry)
- `--restart unless-stopped` — переживает перезагрузку сервера
- Для ускорения можно установить `cryptography`: `pip install cryptography` в Dockerfile
