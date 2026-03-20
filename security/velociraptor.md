# Velociraptor — DFIR и threat hunting платформа

> Источник: Хакер #322, март 2026

## Что это

Open source платформа для Digital Forensics and Incident Response (DFIR).
«Google для твоих хостов» — задаёшь вопрос, сразу получаешь ответ с нужного хоста или всей сети.

Создан Майклом Коэном (Michel Cohen), вырос из идеи Google.
Язык: Go. Репо: https://github.com/Velocidex/velociraptor

## Ключевые концепции

### VQL (Velociraptor Query Language)
Синтаксис похож на SQL:
```sql
SELECT * FROM <plugin> WHERE <condition>
```

Один запрос может собирать данные из:
- файловой системы
- реестра (Windows)
- памяти процессов
- журналов событий (EVTX)
- сетевых соединений

### Артефакт
Центральная сущность — YAML-файл с описанием:
- **что** собирать
- **как** (VQL-запросы)
- **куда** выводить

Готовых артефактов сотни — Artifact Exchange: https://docs.velociraptor.app/exchange/

### Pull-модель
Агент ничего не отправляет сам. Алгоритм:
1. Агент просыпается
2. Опрашивает сервер на наличие задач
3. Выполняет задачи
4. Возвращает результат

Снижает нагрузку на сеть, работает тише.

## Архитектура

```
Velociraptor Server (Web GUI + API)
        │
        │  pull-модель (HTTPS)
        │
   ┌────┴────┐
   │  Агенты  │  ← Linux / Windows / macOS / ARM (Apple Silicon)
   └─────────┘
```

Всё хранится на своём сервере — никаких облачных подписок.

## Установка сервера (Linux)

```bash
# Скачать бинарник с GitHub
wget https://github.com/Velocidex/velociraptor/releases/latest/download/velociraptor-linux-amd64
chmod +x velociraptor-linux-amd64
mv velociraptor-linux-amd64 velociraptor

# Сгенерировать конфиг (интерактивно)
sudo ./velociraptor config generate -i

# Разрешить доступ к GUI из сети (по умолчанию 127.0.0.1)
# В server.config.yaml → секция GUI → исправить 127.0.0.1 на 0.0.0.0

# Запуск сервера в фоне
sudo nohup ./velociraptor --config server.config.yaml frontend > /var/log/velociraptor.log 2>&1 &
```

Веб-интерфейс по умолчанию: `https://<host>:8889/`

## Установка клиентов

### Генерация клиентского конфига
```bash
./velociraptor --config server.config.yaml config client > client.config.yaml
```

### Windows (через BAT/service)
```bat
@echo off
velociraptor.exe --config client.config.yaml service install
sc start VelociraptorClient
```

### Linux (RHEL/Fedora/РЕД ОС)
```bash
sudo ./velociraptor --config client.config.yaml service install
```

### macOS (Apple Silicon — ARM)
Скачивать бинарник для `darwin-arm64`, шаги аналогичны Linux.

## Работа в GUI

### Hunts (Охоты)
Вкладка с иконкой прицела. Создать охоту:
1. `+` → New Hunt
2. Заполнить: название, теги, описание
3. Условие включения: Include Labels (метки хостов)
4. Выбрать артефакты
5. Launch → Start

### Labels (Метки хостов)
Помогают фильтровать хосты для задач:
- по ОС (linux, windows, macos)
- по архитектуре (amd64, arm64)
- по домену / роли

### Просмотр результатов
- **Notebook** — таблицы прямо в браузере
- **Collected** — история выполненных задач на хосте
- **Results** — данные с выбором колонок
- **JSON** — сырой ответ клиента (кнопка бинокля)
- Экспорт в CSV

### Live Shell
Кнопка в правом верхнем углу хоста. Выполняет команды на хосте с правами root.
Полезно когда сервисы упали, но агент ещё живёт. Автодополнения нет — вводить команды целиком.

## Офлайн-коллектор (работа «в полях»)
Когда хост недоступен по сети:
1. Сформировать офлайн-коллектор (значок бумажного самолётика)
2. Выбрать нужные артефакты → скачать .exe или бинарник
3. Запустить на изолированном хосте — создаст архив с артефактами
4. Загрузить архив в Velociraptor → плагин `Server.Utils.ImportCollection`
5. Анализировать как обычный хост (появится «виртуальный клиент»)

## Полезные артефакты по платформам

### Linux

| Артефакт | Что делает |
|---|---|
| `Linux.RHEL.Packages` | Список пакетов (DNF/YUM) |
| `Linux.Sys.Users` | Пользователи из `/etc/passwd` |
| `Linux.Ssh.PrivateKeys` | SSH приватные ключи и их шифрование |
| `Linux.Syslog.SSHLogin` | Все попытки входа по SSH |
| `Linux.Carving.SSHLogs` | Детальный разбор SSH-логов |
| `Linux.Sys.Crontab` | Содержимое crontab — поиск persistence |
| `Linux.Sys.BashHistory` | История команд Bash |
| `Linux.Sys.LastUserLogin` | Разбор WTMP — последние входы |
| `Linux.Detection.MemFD` | Процессы, запущенные из ОЗУ через memfd_create |
| `Linux.Detection.Yara.Process` | YARA-сканирование процессов в памяти |
| `Linux.Detection.IncorrectPermissions` | Файлы с нелегитимными владельцами |
| `Linux.Forensics.EnvironmentVariables` | Persistence через переменные окружения (T1546.004) |
| `Linux.System.PAM` | Аудит `/etc/pam.d/` — подозрительные PAM модули |
| `Linux.Detection.CVE20214034` | Поиск эксплуатации PwnKit (CVE-2021-4034) |

### Windows

| Артефакт | Что делает |
|---|---|
| `Windows.EventLogs.Evtx` | Журналы событий Windows за период |
| `Windows.Timeline.Prefetch` | Prefetch — недавно запускавшиеся приложения |
| `Windows.Forensics.Lnk` | LNK-файлы — источник заражения |
| `Windows.Sys.Processes` | Список процессов с хешами и командными строками |
| `Windows.Registry.RunKeys` | Ключи автозагрузки из реестра |
| `Windows.Sysinternals.Autoruns` | Все точки автозагрузки (через Autoruns) |
| `Windows.Detection.Yara.*` | YARA-сканирование файлов и памяти |
| `Windows.EventLogs.PowerShell` | События PowerShell — поиск вредоносных скриптов |
| `Windows.EventLogs.RDPAuth` | История RDP-подключений |

### macOS

| Артефакт | Что делает |
|---|---|
| `MacOS.System.Users` | Локальные пользователи, UID, домашние директории |
| `MacOS.System.Plist` | LaunchAgents/Daemons — persistence |
| `MacOS.System.Wifi` | SSID и даты подключения к Wi-Fi (геолокация) |
| `MacOS.Applications.MRU` | Недавно открытые документы |
| `MacOS.Applications.Chrome.History` | История Chrome |
| `MacOS.Network.Netstat` | Текущие соединения, открытые порты |

## Плюсы / Минусы

**Плюсы:**
- Targeted acquisition — берёшь только нужное, а не весь образ диска
- Скорость: 20 секунд вместо 4 часов на образ
- Pull-модель — тихая работа, не тревожит пользователя
- Данные остаются у тебя на сервере
- Огромная библиотека готовых артефактов (Artifact Exchange)
- Live shell — последний рубеж когда сервис упал
- Мониторинг в реальном времени + алерты
- Экспорт в JSON → SIEM/SOAR

**Минусы:**
- Порог входа: нужно знать VQL и понимать СУБД
- Ты сам отвечаешь за инфраструктуру (бэкапы, TLS, нагрузка)
- Не коробочное решение — нужно активно использовать

## Интеграция с Elastic
Velociraptor умеет отправлять результаты в Elasticsearch.
Конфиг в server.config.yaml → секция `Monitoring` → Elastic output.

## Ссылки

- GitHub: https://github.com/Velocidex/velociraptor
- Docs: https://docs.velociraptor.app/
- Artifact Exchange: https://docs.velociraptor.app/exchange/
- Releases: https://github.com/Velocidex/velociraptor/releases
