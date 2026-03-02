# Хардинг Linux-сервера

> Источник: [How-To-Secure-A-Linux-Server](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server) — живой гайд, 6+ лет, активно развивается сообществом.  
> Дополнительно: [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks/) — индустриальный стандарт, перекрывает этот гайд по строгости.

---

## Принципы перед стартом

Прежде чем что-то делать — определи **threat model**:

- Зачем нужна безопасность? Что защищаешь?
- Какие векторы атаки реальны именно для тебя?
  - Физический доступ к серверу?
  - Открытые порты наружу?
  - Внутренняя сеть с доверенными машинами?
- Готов ли ты к ситуации, когда сам себя заблокируешь?

Чем чётче ответы — тем адекватнее конфигурация. Излишняя безопасность = неудобство без реального профита.

---

## 1. SSH — первоочередной приоритет

### 1.1 Ключи вместо паролей

```bash
# На клиентской машине
ssh-keygen -t ed25519

# Скопировать публичный ключ на сервер
ssh-copy-id user@server
```

Ed25519 — лучший выбор: эллиптическая криптография, лучше ECDSA/RSA по безопасности и производительности.

### 1.2 Группа для SSH-доступа

Позволяет точечно управлять тем, кто вообще может зайти:

```bash
sudo groupadd sshusers
sudo usermod -a -G sshusers username
```

### 1.3 `/etc/ssh/sshd_config` — ключевые настройки

```
# Только современный протокол
Protocol 2

# Только нужные ключевые алгоритмы
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key
KexAlgorithms curve25519-sha256@libssh.org,ecdh-sha2-nistp521,...
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,...
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,...

# Пускать только из нашей группы
AllowGroups sshusers

# Root не логинится напрямую
PermitRootLogin no

# Только ключи, без паролей
PasswordAuthentication no
PermitEmptyPasswords no

# Таймауты и лимиты
LoginGraceTime 30
MaxAuthTries 2
MaxSessions 2
MaxStartups 2
ClientAliveInterval 300
ClientAliveCountMax 0

# Отключить всё лишнее
X11Forwarding no
AllowTcpForwarding no
AllowStreamLocalForwarding no
GatewayPorts no
PermitTunnel no
AllowAgentForwarding no
Compression no
TCPKeepAlive no

# Не читать .rhosts
IgnoreRhosts yes
HostbasedAuthentication no

# Логировать fingerprint ключа при входе — нужно для аудита
LogLevel VERBOSE

# SFTP через internal-sftp (изолированный)
Subsystem sftp internal-sftp -f AUTHPRIV -l INFO
```

**Проверка конфига без рестарта:**
```bash
sudo sshd -T
```

**Найти дублирующиеся настройки (SSH не любит дубли):**
```bash
awk 'NF && $1!~/^(#|HostKey)/{print $1}' /etc/ssh/sshd_config | sort | uniq -c | grep -v ' 1 '
```

### 1.4 Удалить слабые Diffie-Hellman ключи

```bash
# Оставить только ключи >= 3072 бит
sudo cp /etc/ssh/moduli /etc/ssh/moduli.bak
sudo awk '$5 >= 3071' /etc/ssh/moduli | sudo tee /etc/ssh/moduli.tmp
sudo mv /etc/ssh/moduli.tmp /etc/ssh/moduli
```

### 1.5 2FA для SSH (опционально, но хорошо)

```bash
sudo apt install libpam-google-authenticator

# Настроить для нужного пользователя (НЕ от root)
google-authenticator
```

Добавить в `/etc/pam.d/sshd`:
```
auth required pam_google_authenticator.so nullok
```

В `sshd_config`:
```
ChallengeResponseAuthentication yes
```

> **Важно:** держи 2-й терминал открытым перед любыми изменениями SSH — иначе рискуешь заблокировать себя.

---

## 2. Базовая безопасность системы

### 2.1 Ограничить sudo

```bash
sudo groupadd sudousers
sudo usermod -a -G sudousers username

# В /etc/sudoers (через visudo!)
%sudousers   ALL=(ALL:ALL) ALL
```

### 2.2 Ограничить su

```bash
sudo groupadd suusers
sudo usermod -a -G suusers username
sudo dpkg-statoverride --update --add root suusers 4750 /bin/su
```

### 2.3 Автообновления безопасности

На Debian/Ubuntu:
```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

Логи смотреть в `/var/log/unattended-upgrades/`.

### 2.4 Защита /proc

Добавить в `/etc/fstab`:
```
proc /proc proc defaults,hidepid=2 0 0
```

Скрывает процессы других пользователей из `/proc`.

### 2.5 Сложные пароли (если пароли используются)

```bash
sudo apt install libpam-pwquality
```

Настроить в `/etc/security/pwquality.conf`.

---

## 3. Сеть и файрвол

### 3.1 UFW — простой файрвол

```bash
sudo apt install ufw

# Сначала разрешить SSH, потом включать!
sudo ufw allow ssh
sudo ufw allow 22/tcp  # или свой порт

# Разрешить только нужное
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

sudo ufw enable
sudo ufw status verbose
```

**⚠️ Правило:** сначала открой SSH, потом `ufw enable` — иначе отрежешь себя.

### 3.2 Fail2Ban — защита от брутфорса

```bash
sudo apt install fail2ban
```

Базовый конфиг `/etc/fail2ban/jail.local`:
```ini
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
port    = ssh
logpath = /var/log/auth.log
maxretry = 3
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Статус и заблокированные IP
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

### 3.3 CrowdSec — современная альтернатива Fail2Ban

Более умный инструмент с коллективным intelligence (блокирует известные bad-actor IP):
- [crowdsec.net](https://crowdsec.net)
- Работает как Fail2Ban, но знает о глобальных угрозах

### 3.4 PSAD — обнаружение сканирования портов

```bash
sudo apt install psad
```

Интегрируется с iptables, детектирует port scan и другие подозрительные активности.

---

## 4. Аудит и мониторинг

### 4.1 Lynis — аудит безопасности системы

```bash
sudo apt install lynis
sudo lynis audit system
```

Даёт score и конкретные рекомендации. Запускать периодически.

### 4.2 Открытые порты

```bash
# Что слушает сервер
ss -tlnp
ss -tlnpu  # включая UDP

# Альтернатива
sudo netstat -tlnp
```

### 4.3 logwatch — дайджест логов на почту

```bash
sudo apt install logwatch
sudo logwatch --output stdout --detail high --range today
```

### 4.4 AIDE — целостность файлов

File Integrity Monitoring — замечает изменения в системных файлах:
```bash
sudo apt install aide
sudo aideinit
sudo aide --check
```

### 4.5 Rkhunter — поиск руткитов

```bash
sudo apt install rkhunter
sudo rkhunter --update
sudo rkhunter --check
```

---

## 5. Инструменты для аудита конфигурации

| Инструмент | Что делает |
|---|---|
| **Lynis** | Аудит безопасности системы, даёт конкретные советы |
| **AIDE** | Мониторинг целостности файлов |
| **Rkhunter** | Поиск руткитов и backdoor |
| **ClamAV** | Антивирус (для серверов, принимающих файлы от пользователей) |
| **OSSEC** | Host Intrusion Detection System (HIDS) |
| **ss / netstat** | Аудит открытых портов |

---

## 6. Чеклист первого входа на новый сервер

```bash
# 1. Обновить систему
sudo apt update && sudo apt upgrade -y

# 2. Убедиться что SSH работает до изменений (2 терминала!)

# 3. Создать пользователя, добавить в sudousers, sshusers
sudo adduser username
sudo usermod -a -G sudo,sshusers username

# 4. Скопировать SSH-ключ
ssh-copy-id username@server

# 3. Настроить sshd_config (PermitRootLogin no, PasswordAuthentication no...)
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo nano /etc/ssh/sshd_config
sudo sshd -T  # проверить
sudo systemctl restart sshd

# 4. UFW
sudo ufw allow ssh && sudo ufw enable

# 5. Fail2Ban
sudo apt install fail2ban

# 6. Удалить слабые DH ключи
sudo awk '$5 >= 3071' /etc/ssh/moduli | sudo tee /etc/ssh/moduli.tmp && sudo mv /etc/ssh/moduli.tmp /etc/ssh/moduli

# 7. Lynis — посмотреть что ещё можно улучшить
sudo apt install lynis && sudo lynis audit system
```

---

## Подводные камни

- **Сначала SSH, потом UFW** — включение `ufw enable` без правила для SSH = потеря доступа
- **Всегда держи 2-й терминал** при изменении sshd_config
- **Дублирующиеся настройки в sshd_config** — SSH берёт первую, игнорирует вторую. Могут быть сюрпризы
- **`PasswordAuthentication no`** нельзя включать, пока не убедился что ключ работает
- **hidepid=2 в /proc** может сломать некоторые системные утилиты — проверяй после применения
- **Автообновления** могут ломать конфигурации — тестируй в dev окружении

---

## Ресурсы

- [How-To-Secure-A-Linux-Server](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server) — основной гайд
- [How-To-Secure-A-Linux-Server-With-Ansible](https://github.com/moltenbit/How-To-Secure-A-Linux-Server-With-Ansible) — Ansible-плейбуки для автоматизации
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks/) — индустриальный стандарт хардинга
- [Mozilla OpenSSH Guidelines](https://infosec.mozilla.org/guidelines/openssh) — настройки sshd от Mozilla
- [Arch Linux Security Wiki](https://wiki.archlinux.org/index.php/Security)
