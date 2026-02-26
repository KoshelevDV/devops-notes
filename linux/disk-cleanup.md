# Linux: очистка места — логи, кэши, Docker, без поломок

> **Дата:** 2026-02-26  
> **Теги:** `linux` `disk` `logs` `journald` `docker` `cleanup`

---

## 🎯 Проблема

Диск заполнился. Нужно освободить место быстро и безопасно, не сломав систему.

---

## 0. Разведка — где место вообще кончилось

```bash
# Общий обзор по разделам
df -h

# Что жрёт место в конкретной папке (топ-20)
du -ah /var | sort -rh | head -20
du -ah /opt | sort -rh | head -20

# Найти файлы >100 МБ по всей системе
find / -xdev -size +100M -printf "%s\t%p\n" 2>/dev/null | sort -rn | head -20

# Найти файлы >1 ГБ
find / -xdev -size +1G -ls 2>/dev/null
```

---

## 1. Journald (systemd логи) — самое безопасное

```bash
# Посмотреть сколько занимают логи прямо сейчас
journalctl --disk-usage

# Оставить только последние 200 МБ (удаляет старые по ротации)
sudo journalctl --vacuum-size=200M

# Оставить логи только за последние 2 недели
sudo journalctl --vacuum-time=2weeks

# Оставить только последние 7 дней и не более 500 МБ
sudo journalctl --vacuum-time=7d --vacuum-size=500M
```

Чтобы ограничение сохранялось после перезагрузки — прописать в конфиг:
```bash
sudo nano /etc/systemd/journald.conf
```
```ini
[Journal]
SystemMaxUse=500M        # максимум на диске
SystemKeepFree=1G        # всегда оставлять свободными
MaxRetentionSec=2week    # хранить не дольше
```
```bash
sudo systemctl restart systemd-journald
```

---

## 2. Обычные файлы логов в /var/log

```bash
# Посмотреть что тяжёлое
du -ah /var/log | sort -rh | head -20

# Безопасно очистить конкретный лог (не удалять! — просто обнулить)
sudo truncate -s 0 /var/log/nginx/access.log
sudo truncate -s 0 /var/log/nginx/error.log

# Ротация логов вручную (штатный механизм, самый правильный способ)
sudo logrotate -f /etc/logrotate.conf

# Найти старые ротированные архивы и удалить
sudo find /var/log -name "*.gz" -mtime +30 -delete
sudo find /var/log -name "*.1" -mtime +7 -delete
```

⚠️ **Никогда не делать `rm /var/log/something.log`** если процесс держит файл открытым —
место не освободится, а приложение начнёт писать в никуда. Только `truncate -s 0`.

Проверить открытые удалённые файлы (место занято, но файл удалён):
```bash
sudo lsof | grep deleted | sort -k7 -rn | head -10
# Если видишь такое — перезапусти соответствующий процесс
```

---

## 3. Docker — самый большой пожиратель

```bash
# Посмотреть сколько занимает docker
docker system df

# Убрать всё неиспользуемое (образы без контейнеров, stopped контейнеры, сети)
# БЕЗОПАСНО — не трогает запущенные контейнеры и их volumes
docker system prune

# То же самое + volumes (⚠️ ОСТОРОЖНО — удаляет данные!)
docker system prune --volumes

# Только неиспользуемые образы
docker image prune -a

# Только остановленные контейнеры
docker container prune

# Только анонимные volumes (без имени)
docker volume prune

# Только неиспользуемые сети
docker network prune
```

Логи конкретного контейнера:
```bash
# Посмотреть размер
du -h $(docker inspect --format='{{.LogPath}}' <container_name>)

# Обнулить (безопасно, контейнер продолжает работать)
sudo truncate -s 0 $(docker inspect --format='{{.LogPath}}' <container_name>)

# Или настроить ограничение в docker-compose.yml (правильный путь):
services:
  app:
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "3"
```

Глобально для всех новых контейнеров — в `/etc/docker/daemon.json`:
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```
```bash
sudo systemctl reload docker
```

---

## 4. Пакетный менеджер (apt/dnf)

```bash
# Debian/Ubuntu
sudo apt autoremove          # удалить ненужные зависимости
sudo apt clean               # очистить кэш .deb файлов
sudo apt autoclean           # удалить только устаревшие пакеты из кэша

# Fedora/RHEL
sudo dnf autoremove
sudo dnf clean all           # кэш пакетов
sudo dnf clean packages      # только .rpm файлы
```

---

## 5. Snap / Flatpak — забытые пожиратели

```bash
# Snap — старые версии (snap хранит 2-3 ревизии по умолчанию)
snap list --all | awk '/disabled/{print $1, $3}' | \
  while read name rev; do sudo snap remove "$name" --revision="$rev"; done

# Или глобально: оставить только последнюю версию
sudo snap set system snapshots.automatic.retention=no

# Flatpak
flatpak uninstall --unused
```

---

## 6. Tmpfs и /tmp

```bash
# Очистить /tmp (при запущенной системе — только то что не занято)
sudo find /tmp -atime +1 -delete 2>/dev/null

# /tmp обычно tmpfs и очищается при перезагрузке
mount | grep tmp
```

---

## 7. Старые ядра (Debian/Ubuntu)

```bash
uname -r                        # текущее ядро
dpkg --list 'linux-image*'      # все установленные
sudo apt autoremove             # удалит старые автоматически
```

---

## 8. Кэши пользователя

```bash
# pip
pip cache purge

# npm
npm cache clean --force

# cargo
cargo cache --autoclean         # если установлен cargo-cache
# или вручную:
rm -rf ~/.cargo/registry/cache/

# go modules
go clean -modcache

# ~/.cache — смотри что там
du -ah ~/.cache | sort -rh | head -10
```

---

## 💡 Подводные камни

- **Никогда не `rm` файлы активных логов** — используй `truncate -s 0`. Иначе процесс держит
  fd открытым, место не освобождается.

- **`docker system prune --volumes`** — удалит ВСЕ анонимные volumes, включая БД без имени.
  Всегда проверяй `docker volume ls` перед этим.

- **`df -h` показывает 100%`, но `du` меньше?** — значит есть удалённые файлы с открытыми
  дескрипторами. Найди через `lsof | grep deleted` и перезапусти процесс.

- **`/var/log/journal/` растёт бесконечно** без конфигурации journald. На свежих серверах —
  сразу прописать `SystemMaxUse` в `/etc/systemd/journald.conf`.

- **Snap** хранит 3 ревизии по умолчанию, это может занимать гигабайты.

---

## 🚀 Быстрый рецепт «освободить место прямо сейчас»

```bash
# 1. Разведка
df -h && du -ah /var/log /var/lib/docker 2>/dev/null | sort -rh | head -20

# 2. Логи journald
sudo journalctl --vacuum-size=200M --vacuum-time=2weeks

# 3. Docker (без volumes — безопасно)
docker system prune -f

# 4. Пакеты
sudo apt autoremove && sudo apt clean   # Debian/Ubuntu
# sudo dnf autoremove && sudo dnf clean all  # Fedora

# 5. Проверить результат
df -h
```

---

## 📎 Документация

- [journalctl man page](https://www.freedesktop.org/software/systemd/man/journalctl.html)
- [docker system prune](https://docs.docker.com/engine/reference/commandline/system_prune/)
- [logrotate man page](https://linux.die.net/man/8/logrotate)
