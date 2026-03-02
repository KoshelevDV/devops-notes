# Внутренний CA с Smallstep (step-ca)

> Позволяет выдавать TLS-сертификаты для внутренней инфраструктуры, которым доверяют все машины в сети.  
> Преимущество перед self-signed: один root.crt распространяется по всем тачкам — и все сертификаты от этого CA сразу доверенные.

---

## Установка (Debian/Ubuntu)

```bash
# step-ca (сервер)
wget https://dl.smallstep.com/certificates/docs-ca-install/latest/step-ca_amd64.deb
sudo dpkg -i step-ca_amd64.deb

# step CLI (клиент)
wget https://dl.smallstep.com/cli/docs-ca-install/latest/step-cli_amd64.deb
sudo dpkg -i step-cli_amd64.deb
```

---

## Инициализация CA (один раз)

```bash
# Создать пользователя
sudo useradd --user-group --system --home /etc/step-ca --shell /bin/false step

# Разрешить слушать порт 443 без root
sudo setcap CAP_NET_BIND_SERVICE=+eip $(which step-ca)

# Инициализация — создаст Root CA + Intermediate CA
step ca init
# Задаст вопросы: имя CA, DNS/IP, порт (443), provisioner, пароль

# Переместить конфиг в /etc/step-ca
sudo mkdir -p /etc/step-ca
sudo mv $(step path)/* /etc/step-ca
sudo chown -R step:step /etc/step-ca

# Исправить путь к БД
cat <<< $(jq '.db.dataSource = "/etc/step-ca/db"' /etc/step-ca/config/ca.json) \
  > /etc/step-ca/config/ca.json
```

### systemd-сервис

```ini
# /etc/systemd/system/step-ca.service
[Unit]
Description=step-ca internal CA
After=network.target

[Service]
User=step
Group=step
ExecStart=/usr/bin/step-ca /etc/step-ca/config/ca.json \
  --password-file /etc/step-ca/password.txt
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now step-ca
sudo systemctl status step-ca
```

---

## Получить fingerprint CA

```bash
step ca health
# или при первом запуске step-ca выводит fingerprint в лог
```

---

## Выдать wildcard-сертификат

```bash
# Если step CLI уже сбутстрапирован (см. ниже)
step ca certificate "*.aboba.local" wildcard.crt wildcard.key \
  --san "*.aboba.local" \
  --san "aboba.local" \
  --not-after 87600h   # 10 лет

# Для HAProxy — склеить в один файл
cat wildcard.crt wildcard.key > aboba.local.pem
```

---

## Раздать root CA на клиентские машины

### Bootstrap клиента (сохраняет CA и доверяет ему)

```bash
# Узнать fingerprint на сервере:
step ca root root.crt
# fingerprint покажет при bootstrap или в логах step-ca

# На клиентской машине:
step ca bootstrap \
  --ca-url https://10.0.30.20 \
  --fingerprint <fingerprint>

# Получить root-сертификат
step ca root root.crt
```

### Добавить root.crt в системное доверие

**Debian/Ubuntu:**
```bash
sudo cp root.crt /usr/local/share/ca-certificates/aboba-local.crt
sudo update-ca-certificates
```

**RHEL/Fedora:**
```bash
sudo cp root.crt /etc/pki/ca-trust/source/anchors/aboba-local.crt
sudo update-ca-trust
```

**Windows (через PowerShell, admin):**
```powershell
Import-Certificate -FilePath root.crt -CertStoreLocation Cert:\LocalMachine\Root
```

**macOS:**
```bash
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain root.crt
```

---

## Обновить сертификат (когда истекает)

```bash
# Автоматически (step умеет renew)
step ca renew wildcard.crt wildcard.key --force

# Добавить в cron для автообновления:
# 0 0 1 * * step ca renew /etc/haproxy/certs/aboba.local.pem --force && systemctl reload haproxy
```

---

## Полезные команды

```bash
# Проверить здоровье CA
step ca health

# Список provisioner-ов
step ca provisioner list

# Посмотреть что внутри сертификата
step certificate inspect wildcard.crt

# Проверить доверие
curl https://service.aboba.local   # должен работать без -k
```

---

## Подводные камни

- **Пароль step-ca** — хранится в `/etc/step-ca/password.txt`, без него CA не стартует
- **Fingerprint меняется** при `step ca init` заново — придётся ребутстрапить все клиенты
- **`--not-after 87600h`** — 10 лет (8760h = 1 год, ×10)
- **Порт в ca.json** — при `init` указывается, потом менять через `sudo vim /etc/step-ca/config/ca.json` + restart
- **`jq` для редактирования db.dataSource** — `cat <<<` не работает с sudo, делай через temp-файл если нужен root

---

## Источник

История команд с машины `smallstep` (10.0.30.20)
