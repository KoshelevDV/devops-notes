# Headscale + Tailscale — Self-hosted Mesh VPN

**Headscale GitHub:** https://github.com/juanfont/headscale  
**Tailscale docs:** https://tailscale.com/kb  
**Дата заметки:** 2026-03-29

---

## Что это

**Tailscale** — mesh VPN на базе WireGuard. Каждый узел соединяется с каждым напрямую (P2P), без центрального узла на пути трафика. Control plane только координирует соединения.

**Headscale** — open source реализация Tailscale control server (coordinator). Заменяет `controlplane.tailscale.com`. Клиенты — оригинальные Tailscale.

```
Обычный Tailscale:                Self-hosted Headscale:
  клиенты → tailscale.com           клиенты → твой сервер
  (данные идут P2P через WG)        (данные идут P2P через WG)
  control plane — проприетарный     control plane — open source
```

---

## Архитектура

```
                    ┌─────────────────────┐
                    │  Headscale Server   │
                    │  (control plane)    │
                    │  :8080 (API/gRPC)   │
                    │  :8888 (UI)         │
                    └──────────┬──────────┘
                               │ координация
              ┌────────────────┼────────────────┐
              ↓                ↓                ↓
       ┌──────────┐    ┌──────────┐    ┌──────────┐
       │  Node 1  │◄──►│  Node 2  │◄──►│  Node 3  │
       │ 100.64.x │    │ 100.64.x │    │ 100.64.x │
       └──────────┘    └──────────┘    └──────────┘
              WireGuard P2P (данные не идут через сервер)
```

---

## Установка Headscale (вручную)

```bash
# Скачать бинарник
curl -Lo /usr/local/bin/headscale \
  https://github.com/juanfont/headscale/releases/latest/download/headscale_linux_amd64
chmod +x /usr/local/bin/headscale

# Создать пользователя
useradd --system --no-create-home headscale

# Создать директории
mkdir -p /etc/headscale /var/lib/headscale

# Сгенерировать конфиг
headscale generate private-key > /var/lib/headscale/private.key
```

### Минимальный config.yaml

```yaml
server_url: https://vpn.yourserver.com:8080
listen_addr: 0.0.0.0:8080
grpc_listen_addr: 0.0.0.0:50443

private_key_path: /var/lib/headscale/private.key
noise:
  private_key_path: /var/lib/headscale/noise_private.key

ip_prefixes:
  - 100.64.0.0/10
  - fd7a:115c:a1e0::/48

db_type: sqlite3
db_path: /var/lib/headscale/db.sqlite

dns_config:
  magic_dns: true
  base_domain: vpn.internal
  nameservers:
    - 1.1.1.1
```

### Systemd unit

```ini
[Unit]
Description=Headscale VPN
After=network.target

[Service]
User=headscale
ExecStart=/usr/local/bin/headscale serve
Restart=on-failure
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/var/lib/headscale

[Install]
WantedBy=multi-user.target
```

---

## Управление через CLI

```bash
# Создать пользователя (namespace)
headscale users create myuser

# Создать pre-auth key (для автоматического подключения узлов)
headscale preauthkeys create --user myuser --reusable --expiration 24h
# → hskey-...

# Список узлов
headscale nodes list

# Список пользователей
headscale users list

# Статус конкретного узла
headscale nodes show <node-id>

# Переименовать узел
headscale nodes rename --node <id> <new-name>

# Удалить узел
headscale nodes delete --node <id>

# Управление тегами
headscale nodes tag --node <id> --tags tag:server,tag:prod
```

---

## Подключение клиентов

### Linux

```bash
# Установка Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Подключиться к Headscale (с pre-auth key)
tailscale up \
  --login-server=https://vpn.yourserver.com:8080 \
  --authkey=hskey-...

# Или через браузер (интерактивно)
tailscale up --login-server=https://vpn.yourserver.com:8080
# → откроет ссылку, подтвердить в headscale:
headscale nodes register --user myuser --key <key>
```

### macOS / Windows / iOS / Android

Установить официальный Tailscale → в настройках указать custom control server.

---

## Headscale UI (неофициальный)

```bash
docker run -d \
  --name headscale-ui \
  --network host \
  -e HEADSCALE_URL=http://localhost:8080 \
  -v /opt/headscale-ui:/data \
  ghcr.io/gurucomputing/headscale-ui:latest
```

UI: `http://<server>:8888`

Для авторизации в UI нужен API key:
```bash
headscale apikeys create --expiration 90d
```

---

## Ansible роли (в infra-monitoring)

| Роль | Описание |
|---|---|
| `headscale` | Устанавливает Headscale control server (бинарник + systemd) |
| `headscale-ui` | Деплоит UI через Docker |
| `tailscale-client` | Устанавливает Tailscale и подключает к сети |

```bash
# Развернуть сервер
ansible-playbook ansible/playbooks/headscale.yml \
  -i inventories/production/ \
  --limit headscale_server \
  -e "headscale_server_url=https://vpn.myserver.com:8080"

# Подключить все хосты
ansible-playbook ansible/playbooks/tailscale.yml \
  -i inventories/production/ \
  -e "tailscale_login_server=https://vpn.myserver.com:8080" \
  -e "tailscale_authkey=hskey-..."
```

**ВАЖНО:** `tailscale_authkey` хранить в Vault, не в открытом виде:
```yaml
# group_vars/all.yml
tailscale_authkey: "{{ lookup('hashi_vault', 'secret=infra/tailscale:authkey') }}"
```

---

## ACL политики (для изоляции)

```json
// /etc/headscale/acl.hujson
{
  "groups": {
    "group:servers": ["tag:server"],
    "group:agents": ["tag:agent"]
  },
  "acls": [
    // Агенты могут общаться друг с другом
    {"action": "accept", "src": ["group:agents"], "dst": ["group:agents:*"]},
    // Серверы могут ходить к агентам
    {"action": "accept", "src": ["group:servers"], "dst": ["group:agents:*"]},
    // SSH для adminов
    {"action": "accept", "src": ["myuser@"], "dst": ["*:22"]}
  ]
}
```

---

## Subnet routing (доступ к локальной сети)

Если нужно чтобы через один узел была доступна вся подсеть:

```bash
# На роутере/шлюзе
tailscale up \
  --login-server=https://vpn.myserver.com:8080 \
  --authkey=hskey-... \
  --advertise-routes=10.0.0.0/24

# Включить роуты в headscale
headscale routes list
headscale routes enable --route <route-id>

# На клиентах принять роуты
tailscale up --accept-routes
```

---

## Exit Node (весь трафик через один узел)

```bash
# Объявить exit node
tailscale up --advertise-exit-node

# Включить в headscale
headscale routes enable --route <exit-route-id>

# Использовать exit node на клиенте
tailscale up --exit-node=<node-ip>
```

---

## Ограничения Headscale vs официального Tailscale

| Фича | Headscale | Tailscale |
|---|---|---|
| Mesh WireGuard | ✅ | ✅ |
| MagicDNS | ✅ | ✅ |
| Subnet routing | ✅ | ✅ |
| Exit nodes | ✅ | ✅ |
| ACL policies | ✅ (файл) | ✅ (UI) |
| Tailscale SSH | ✅ | ✅ |
| Taildrop (файлы) | ❌ | ✅ |
| Funnel (публичный URL) | ❌ | ✅ |
| iOS/Android клиент | ✅ (офиц.) | ✅ |
| Web UI | ✅ (неофиц.) | ✅ |

---

## Use case: multi-machine OpenClaw агенты

```yaml
# inventories/production/hosts.yml
headscale_server:
  hosts:
    vps-01:  # control plane

tailscale_nodes:
  hosts:
    home-server:
      tailscale_advertise_tags: ["tag:orchestrator"]
    mac-mini-1:
      tailscale_advertise_tags: ["tag:agent", "tag:coder"]
    mac-mini-2:
      tailscale_advertise_tags: ["tag:agent", "tag:security"]
```

После подключения все машины доступны по:
- `home-server.vpn.internal`
- `mac-mini-1.vpn.internal`
- `mac-mini-2.vpn.internal`

OpenClaw remote gateway настраивается через эти DNS имена.
