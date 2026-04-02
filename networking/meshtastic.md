# Meshtastic — децентрализованная mesh-сеть на LoRa

> Open-source проект для создания mesh-сетей без интернета и GSM.
> Сайт: https://meshtastic.org | GitHub: https://github.com/meshtastic/firmware

---

## Суть

- Устройства общаются по радио (LoRa) напрямую, без инфраструктуры
- Каждый узел ретранслирует пакеты дальше — получается mesh
- Дальность: 2-15 км в городе, до 50+ км на открытой местности
- Текстовые сообщения, GPS-координаты, телеметрия
- Управление через Bluetooth → приложение Meshtastic (Android/iOS)

---

## Частоты

| Регион | Частота |
|---|---|
| Россия / Европа | **868 МГц** (ISM band, без лицензии) |
| США | 915 МГц |
| Альтернатива | 433 МГц |

---

## Роли узлов

| Роль | Описание |
|---|---|
| **CLIENT** | Обычное устройство с телефоном |
| **ROUTER** | Чистый ретранслятор, без активного пользователя (на чердак/столб) |
| **ROUTER_CLIENT** | Гибрид — и ретранслирует и сам участник |
| **REPEATER** | Минимальное потребление, только ретрансляция |

---

## Железо под 868 МГц

| Плата | Цена | Плюсы | Минусы |
|---|---|---|---|
| **Heltec V3** (SX1262, ESP32) | ~2346₽ | Дёшево, OLED, 28 dBm | GPS нет, нужен внешний аккум |
| **Heltec V4** (SX1262, ESP32-S3) | ~2979₽ | Экономичнее, разъём под GPS L76K | GPS всё равно докупать |
| **TTGO T-Beam v1.2** | ~$25-30 | GPS встроен, 18650 | Крупнее, дороже |
| **Heltec MeshPocket** | ~7072₽ | Готов из коробки, 5000 мАч, e-ink, GPS | Цена |
| **RAK WisBlock** | ~$35+ | Модульный, высокое качество | Сложнее в сборке |

### V3 vs V4

- **Радиомодуль:** оба SX1262, V4 чуть лучше чувствительность
- **MCU:** V3 — ESP32, V4 — ESP32-S3 (быстрее, больше памяти)
- **Потребление:** V4 заметно экономичнее в sleep-режиме
- **GPS:** V3 — UART пины вручную, V4 — выделенный разъём (L76K)

---

## Сценарий: общение на поселке

Для компании в радиусе 1-2 км — MQTT-сервер не нужен, устройства видят друг друга напрямую.

**Минимальный набор:**
- Каждому по Heltec V3 (~2346₽) + повербанк
- Один узел на чердак в режиме ROUTER (от сети 24/7) если есть мёртвые зоны
- В приложении создать приватный канал с паролем

---

## MQTT-сервер (для объединения сетей через интернет)

Нужен если хочется объединить несколько mesh-сетей из разных мест через интернет.

```
Поселок mesh (LoRa) ←→ узел с WiFi ←→ MQTT-брокер ←→ узел с WiFi ←→ другой mesh
```

### Поднять Mosquitto (Docker)

```bash
docker run -d --name mosquitto \
  -p 1883:1883 \
  -p 8883:8883 \
  -v ./mosquitto.conf:/mosquitto/config/mosquitto.conf \
  eclipse-mosquitto
```

`mosquitto.conf` минимальный:
```
listener 1883
allow_anonymous false
password_file /mosquitto/config/passwd

listener 8883
certfile /mosquitto/certs/cert.pem
keyfile /mosquitto/certs/key.pem
cafile /mosquitto/certs/ca.pem
```

Создать пользователя:
```bash
docker exec -it mosquitto mosquitto_passwd -c /mosquitto/config/passwd meshtastic
```

### Настройка узла (Meshtastic app)

Settings → Module Config → MQTT:
- Enabled: ✅
- Server address: `193.124.92.78` (или домен)
- Username / Password
- Encryption enabled: ✅ (рекомендуется)
- JSON enabled: опционально

### Публичный брокер (без своего сервера)

`mqtt.meshtastic.org` — официальный публичный брокер.
Минус: все сообщения идут через серверы Meshtastic.

---

## Топики MQTT

```
msh/RU/2/e/LongFast/!<node_id>   — зашифрованные пакеты
msh/RU/2/json/LongFast/!<node_id> — JSON (если включён JSON output)
```

---

## Подводные камни

- Heltec V3/V4 без GPS — координат на карте не будет у тех у кого нет GPS-модуля
- LoRa — медленный протокол, не для файлов/голоса, только текст и телеметрия
- При плохой антенне дальность резко падает — стоковые антенны слабые, лучше докупить 868 МГц 3-5 dBi
- `allow_anonymous true` в Mosquitto — не делать на публичном IP
- Mosquitto по умолчанию без TLS — нужен 8883 с сертификатом если брокер смотрит в интернет

---

## Ссылки

- Документация: https://meshtastic.org/docs/
- MQTT конфиг: https://meshtastic.org/docs/configuration/module/mqtt/
- Приложение Android: https://play.google.com/store/apps/details?id=com.geeksville.mesh
- Купить Heltec V3: Aliexpress / российские магазины (~2346₽)
