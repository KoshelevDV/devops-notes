# Управление правами доступа — PostgreSQL, MongoDB, ClickHouse, Kubernetes, Linux VM, AD/LDAP

## Общие принципы (везде одинаковые)

- **Least Privilege** — минимально необходимые права, не больше
- **Separation of Duties** — разные роли для разных задач (readonly, readwrite, admin)
- **Service accounts** — отдельный пользователь на каждый сервис, никогда не `root`/`admin`
- **No shared credentials** — у каждого сервиса/человека свои учётки
- **Rotation** — пароли и токены меняются по расписанию
- **Audit** — логирование всех действий с привилегированными учётками

---

## PostgreSQL

### Встроенные роли

```sql
-- Суперпользователь (создаётся при инициализации)
-- Никогда не использовать для приложений!
\du postgres

-- Посмотреть все роли
\du
-- или
SELECT rolname, rolsuper, rolcreatedb, rolcreaterole, rolcanlogin
FROM pg_roles ORDER BY rolname;
```

### Создание ролей по принципу наименьших прав

```sql
-- 1. Роль только для чтения
CREATE ROLE readonly_role;
GRANT CONNECT ON DATABASE mydb TO readonly_role;
GRANT USAGE ON SCHEMA public TO readonly_role;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_role;
-- Для будущих таблиц:
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO readonly_role;

-- 2. Роль для приложения (CRUD)
CREATE ROLE app_role;
GRANT CONNECT ON DATABASE mydb TO app_role;
GRANT USAGE ON SCHEMA public TO app_role;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_role;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO app_role;

-- 3. Роль для миграций (создание таблиц)
CREATE ROLE migration_role;
GRANT CONNECT ON DATABASE mydb TO migration_role;
GRANT CREATE ON SCHEMA public TO migration_role;
GRANT ALL ON ALL TABLES IN SCHEMA public TO migration_role;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO migration_role;

-- 4. Создать пользователей и назначить роли
CREATE USER app_user WITH PASSWORD 'strong-password' LOGIN;
GRANT app_role TO app_user;

CREATE USER readonly_user WITH PASSWORD 'strong-password' LOGIN;
GRANT readonly_role TO readonly_user;

CREATE USER migration_user WITH PASSWORD 'strong-password' LOGIN;
GRANT migration_role TO migration_user;
```

### Row-Level Security (доступ к строкам)

```sql
-- Включить RLS на таблице
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Политика: пользователь видит только свои заказы
CREATE POLICY user_orders ON orders
    USING (user_id = current_setting('app.current_user_id')::int);

-- Политика для разных ролей
CREATE POLICY admin_all ON orders
    TO admin_role
    USING (true);   -- всё видно

CREATE POLICY user_own ON orders
    TO app_role
    USING (user_id = current_user::int);
```

### Ограничение по сети — pg_hba.conf

```
# TYPE   DATABASE    USER            ADDRESS          METHOD
# Локально только для postgres
local    all         postgres                         peer
# Приложение — только с определённого IP
host     mydb        app_user        10.0.30.0/24     scram-sha-256
# Readonly — только из офисной сети
host     mydb        readonly_user   192.168.1.0/24   scram-sha-256
# Запрет всем остальным
host     all         all             0.0.0.0/0        reject
```

### LDAP-аутентификация

```
# pg_hba.conf
host    all     all     10.0.0.0/8    ldap
    ldapserver=ldap.company.com
    ldapbasedn="dc=company,dc=com"
    ldapsearchattribute=uid
    ldapbinddn="cn=pgbind,dc=company,dc=com"
    ldapbindpasswd="bind-password"
```

```sql
-- Создать роль под LDAP-пользователя (пароль не нужен)
CREATE ROLE "john.doe@company.com" LOGIN;
GRANT readonly_role TO "john.doe@company.com";
```

### Аудит действий

```sql
-- pgaudit расширение
CREATE EXTENSION pgaudit;

-- postgresql.conf
pgaudit.log = 'write, ddl, role'
pgaudit.log_relation = on
pgaudit.log_catalog = off
```

---

## MongoDB

### Включить авторизацию

```yaml
# /etc/mongod.conf
security:
  authorization: enabled
```

### Встроенные роли

| Роль | Что может |
|------|-----------|
| `read` | Только чтение коллекций |
| `readWrite` | Чтение и запись |
| `dbAdmin` | Управление схемой (индексы, stats), без данных |
| `userAdmin` | Управление пользователями БД |
| `dbOwner` | readWrite + dbAdmin + userAdmin |
| `clusterMonitor` | Мониторинг кластера |
| `clusterAdmin` | Полное управление кластером |
| `root` | Суперпользователь |

### Создание пользователей

```javascript
// Подключиться как admin
use admin

// Создать суперпользователя (только для управления)
db.createUser({
  user: "mongoadmin",
  pwd: "strong-password",
  roles: [{ role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase"]
})

// Пользователь для приложения
use mydb
db.createUser({
  user: "app_user",
  pwd: "strong-password",
  roles: [{ role: "readWrite", db: "mydb" }]
})

// Только чтение
db.createUser({
  user: "readonly_user",
  pwd: "strong-password",
  roles: [{ role: "read", db: "mydb" }]
})

// Мониторинг (только метрики, без данных)
use admin
db.createUser({
  user: "monitoring",
  pwd: "strong-password",
  roles: [
    { role: "clusterMonitor", db: "admin" },
    { role: "read", db: "local" }
  ]
})
```

### Custom Role — тонкие права

```javascript
use mydb
db.createRole({
  role: "ordersReadOnly",
  privileges: [
    {
      resource: { db: "mydb", collection: "orders" },
      actions: ["find"]
    }
  ],
  roles: []
})

db.createUser({
  user: "analytics_user",
  pwd: "password",
  roles: [{ role: "ordersReadOnly", db: "mydb" }]
})
```

### LDAP-аутентификация (MongoDB Enterprise / Community 4.4+)

```yaml
# mongod.conf
security:
  authorization: enabled
  ldap:
    servers: "ldap.company.com"
    transportSecurity: tls
    bind:
      method: simple
      queryUser: "cn=mongobind,dc=company,dc=com"
      queryPassword: "bind-password"
    userToDNMapping:
      '[
        {
          match: "(.+)",
          ldapQuery: "dc=company,dc=com??sub?(uid={0})"
        }
      ]'
    authz:
      queryTemplate: "{USER}?memberOf?base"

setParameter:
  authenticationMechanisms: PLAIN
```

### Ограничение по сети

```yaml
# mongod.conf
net:
  bindIp: 127.0.0.1,10.0.30.18   # только нужные интерфейсы
  port: 27017
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongo.pem
    CAFile: /etc/ssl/ca.pem
```

---

## ClickHouse

### Два подхода к управлению пользователями

**1. Конфиг-файлы (классический):**
```xml
<!-- /etc/clickhouse-server/users.d/users.xml -->
<clickhouse>
  <users>
    <!-- Суперпользователь (только для администрирования) -->
    <admin>
      <password_sha256_hex><!-- echo -n 'password' | sha256sum --></password_sha256_hex>
      <networks><ip>::/0</ip></networks>
      <profile>default</profile>
      <quota>default</quota>
    </admin>

    <!-- Пользователь для приложения -->
    <app_user>
      <password_sha256_hex>abc123...</password_sha256_hex>
      <networks>
        <ip>10.0.30.0/24</ip>   <!-- только из внутренней сети -->
      </networks>
      <profile>app_profile</profile>
      <quota>app_quota</quota>
      <allow_databases>
        <database>mydb</database>
      </allow_databases>
    </app_user>

    <!-- Readonly пользователь -->
    <readonly_user>
      <password_sha256_hex>def456...</password_sha256_hex>
      <networks><ip>10.0.0.0/8</ip></networks>
      <profile>readonly</profile>
      <quota>default</quota>
    </readonly_user>
  </users>

  <profiles>
    <readonly>
      <readonly>1</readonly>     <!-- запрет на запись -->
    </readonly>
    <app_profile>
      <readonly>0</readonly>
      <max_memory_usage>10000000000</max_memory_usage>
      <max_execution_time>300</max_execution_time>
    </app_profile>
  </profiles>
</clickhouse>
```

**2. SQL RBAC (с версии 20.x, рекомендуется):**
```sql
-- Включить SQL-управление пользователями
-- config.xml: <access_management>1</access_management> для пользователя default

-- Создать роли
CREATE ROLE readonly_role;
CREATE ROLE app_role;
CREATE ROLE analytics_role;

-- Назначить права на роли
GRANT SELECT ON mydb.* TO readonly_role;
GRANT SELECT, INSERT, ALTER UPDATE, ALTER DELETE ON mydb.* TO app_role;
GRANT SELECT ON mydb.orders TO analytics_role;
GRANT SELECT ON mydb.products TO analytics_role;

-- Создать пользователей
CREATE USER app_user IDENTIFIED WITH sha256_password BY 'strong-password'
    HOST IP '10.0.30.0/24';

CREATE USER readonly_user IDENTIFIED WITH sha256_password BY 'password'
    HOST IP '10.0.0.0/8';

-- Назначить роли
GRANT app_role TO app_user;
GRANT readonly_role TO readonly_user;

-- Посмотреть права
SHOW GRANTS FOR app_user;
```

### LDAP-аутентификация

```xml
<!-- config.xml -->
<ldap_servers>
  <company_ldap>
    <host>ldap.company.com</host>
    <port>636</port>
    <enable_tls>yes</enable_tls>
    <bind_dn>uid={user_name},ou=people,dc=company,dc=com</bind_dn>
    <user_dn_detection>
      <base_dn>dc=company,dc=com</base_dn>
      <search_filter>(&amp;(objectClass=posixAccount)(uid={user_name}))</search_filter>
    </user_dn_detection>
  </company_ldap>
</ldap_servers>
```

```sql
-- Создать пользователя с LDAP-аутентификацией
CREATE USER john_doe IDENTIFIED WITH ldap SERVER 'company_ldap';
GRANT readonly_role TO john_doe;
```

### Квоты и лимиты

```sql
-- Ограничить потребление ресурсов
CREATE QUOTA app_quota FOR INTERVAL 1 hour
    MAX queries = 1000,
    MAX read_rows = 1000000000,
    MAX execution_time = 300
TO app_role;
```

---

## Kubernetes RBAC

### Концепция

```
ServiceAccount / User / Group
        │
        ▼
   RoleBinding / ClusterRoleBinding
        │
        ▼
   Role / ClusterRole
        │
        ▼
   Resources + Verbs
```

### Встроенные ClusterRole

| ClusterRole | Кому давать |
|-------------|------------|
| `view` | Readonly доступ ко всем ресурсам namespace |
| `edit` | Создание/изменение ресурсов, без RBAC |
| `admin` | Полные права в namespace, включая RBAC |
| `cluster-admin` | Полные права на весь кластер (только операторам) |

### Пример: права для команды разработки

```yaml
# Namespace для команды
apiVersion: v1
kind: Namespace
metadata:
  name: team-backend
---
# Роль: деплой приложений без изменения RBAC
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: team-backend
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/exec", "services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]       # читать можно, создавать — нет
---
# Биндинг для группы разработчиков
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-team-deployer
  namespace: team-backend
subjects:
  - kind: Group
    name: "backend-team"          # группа из OIDC/LDAP
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

### ServiceAccount для приложений

```yaml
# Отдельный ServiceAccount — не использовать default!
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
automountServiceAccountToken: false  # не монтировать токен если не нужен
---
# Минимальные права для приложения (например, читать ConfigMap)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["myapp-config"]   # только конкретный ConfigMap
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-rolebinding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: myapp-sa
    namespace: production
roleRef:
  kind: Role
  name: myapp-role
  apiGroup: rbac.authorization.k8s.io
```

### OIDC-интеграция с Dex (LDAP/AD → K8s)

Dex — OIDC-провайдер, который проксирует LDAP/AD для Kubernetes.

```yaml
# dex config.yaml
issuer: https://dex.company.com

connectors:
  - type: ldap
    id: ldap
    name: Company LDAP
    config:
      host: ldap.company.com:636
      insecureSkipVerify: false
      rootCAData: <base64-ca-cert>
      bindDN: cn=dexbind,dc=company,dc=com
      bindPW: bind-password
      userSearch:
        baseDN: ou=people,dc=company,dc=com
        filter: "(objectClass=person)"
        username: uid
        idAttr: DN
        emailAttr: mail
        nameAttr: cn
      groupSearch:
        baseDN: ou=groups,dc=company,dc=com
        filter: "(objectClass=groupOfNames)"
        userMatchers:
          - userAttr: DN
            groupAttr: member
        nameAttr: cn

staticClients:
  - id: kubernetes
    secret: kubernetes-dex-secret
    name: Kubernetes
    redirectURIs:
      - https://kubectl.company.com/callback
```

```yaml
# kube-apiserver — добавить OIDC параметры
- --oidc-issuer-url=https://dex.company.com
- --oidc-client-id=kubernetes
- --oidc-username-claim=email
- --oidc-groups-claim=groups
- --oidc-ca-file=/etc/kubernetes/pki/dex-ca.pem
```

```yaml
# Биндинг по LDAP-группе
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: devops-cluster-admin
subjects:
  - kind: Group
    name: "devops-team"          # LDAP группа
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### Аудит прав

```bash
# Что может конкретный пользователь/SA
kubectl auth can-i --list --as=user:john.doe --as-group=backend-team -n production

# Может ли SA сделать действие
kubectl auth can-i get pods --as=system:serviceaccount:production:myapp-sa -n production

# Kube RBAC Proxy / rbac-lookup
kubectl rbac-lookup john.doe -k user -o wide
```

---

## Linux VM — разграничение прав

### Принципы

```bash
# Никогда не работать под root
# Создать отдельного пользователя для каждого сервиса
useradd -r -s /sbin/nologin -d /var/lib/myapp myapp    # системный пользователь
useradd -r -s /sbin/nologin postgres                    # пример для PostgreSQL
```

### Sudoers — точечные права

```bash
# /etc/sudoers.d/deployer
# Группа deployers может перезапускать только нужные сервисы
%deployers ALL=(ALL) NOPASSWD: /bin/systemctl restart nginx, \
                               /bin/systemctl restart myapp, \
                               /bin/systemctl status *

# Пользователь backup может только читать /etc/
backup-user ALL=(ALL) NOPASSWD: /bin/cat /etc/*, /bin/ls /etc/*

# Запрет на su/bash/sh через sudo
john ALL=(ALL) ALL, !/bin/su, !/bin/bash, !/bin/sh
```

### SSH — ключи и ограничения

```
# /etc/ssh/sshd_config

PermitRootLogin no                   # root только через su
PasswordAuthentication no            # только ключи
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# Разрешить только конкретным группам
AllowGroups sshusers devops

# Запретить форвардинг для ограниченных пользователей
Match Group developers
    AllowTcpForwarding no
    X11Forwarding no
    ForceCommand /usr/local/bin/git-shell   # только git

# Jump-host паттерн: разрешить только проброску на внутренние хосты
Match User jumpuser
    PermitOpen 10.0.30.0/24:22
    ForceCommand echo "Jump host only"
```

### ACL — тонкие права на файлы

```bash
# Посмотреть ACL
getfacl /var/log/app/

# Дать пользователю monitoring право читать логи без добавления в группу
setfacl -m u:monitoring:r /var/log/app/app.log
setfacl -d -m u:monitoring:r /var/log/app/   # для новых файлов

# Группе auditors — читать все конфиги приложения
setfacl -R -m g:auditors:r-x /etc/myapp/
```

### PAM / SSSD — интеграция с AD/LDAP

```bash
# Установка
dnf install sssd sssd-ldap oddjob-mkhomedir    # Fedora/RHEL
apt install sssd sssd-ldap libpam-sss          # Debian/Ubuntu

# /etc/sssd/sssd.conf
[sssd]
services = nss, pam, sudo, ssh
domains = company.com

[domain/company.com]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldaps://ldap.company.com
ldap_search_base = dc=company,dc=com
ldap_tls_reqcert = demand
ldap_tls_cacert = /etc/ssl/certs/company-ca.pem
ldap_default_bind_dn = cn=sssd-bind,dc=company,dc=com
ldap_default_authtok = bind-password

# Автоматически создавать домашние директории
ldap_user_home_directory = /home/%u
enumerate = false   # не качать всех пользователей (безопасно)
```

```bash
# Ограничить вход только для членов группы
# /etc/sssd/sssd.conf
[domain/company.com]
access_provider = ldap
ldap_access_filter = (memberOf=cn=linux-users,ou=groups,dc=company,dc=com)
```

```bash
# /etc/sudoers.d/ldap-groups
# LDAP-группа получает sudo
%devops-team ALL=(ALL) ALL
%ops-team ALL=(ALL) NOPASSWD: /bin/systemctl
```

---

## AD / LDAP — единая точка управления

### Структура (пример)

```
dc=company,dc=com
├── ou=people
│   ├── uid=john.doe
│   ├── uid=jane.smith
│   └── ...
└── ou=groups
    ├── cn=devops-team    → sudo, K8s cluster-admin, DB admin
    ├── cn=backend-team   → K8s deployer, DB app_user
    ├── cn=readonly       → K8s view, DB readonly
    └── cn=linux-users    → SSH доступ к серверам
```

### Что к чему маппить

| LDAP-группа | PostgreSQL | MongoDB | ClickHouse | K8s | Linux |
|-------------|-----------|---------|------------|-----|-------|
| `devops-team` | superuser | root | admin | cluster-admin | sudo ALL |
| `backend-team` | app_role | readWrite | app_role | deployer | sudo systemctl |
| `readonly` | readonly_role | read | readonly | view | нет sudo |
| `dba-team` | pg_monitor + ddl | dbAdmin | admin | — | sudo pg_* |

### OpenLDAP — само-хостинг (если нет AD)

```bash
helm repo add helm-openldap https://jp-gouin.github.io/helm-openldap/
helm install openldap helm-openldap/openldap-stack-ha \
  --namespace ldap --create-namespace \
  --set global.adminPassword=admin \
  --set global.configPassword=config \
  --set phpldapadmin.enabled=true    # веб-интерфейс управления
```

### Проверка подключений

```bash
# Тест LDAP-запроса
ldapsearch -H ldaps://ldap.company.com \
  -D "cn=testbind,dc=company,dc=com" -w password \
  -b "ou=people,dc=company,dc=com" \
  "(uid=john.doe)" cn mail memberOf

# Тест SSSD
id john.doe
getent passwd john.doe
getent group devops-team

# Тест через sssd
sssctl user-checks john.doe
```

---

## Vault (HashiCorp) — централизованное хранение секретов

Если нужно управлять паролями/ротацией сразу для всех систем:

```bash
helm install vault hashicorp/vault \
  --namespace vault --create-namespace

# Динамические пароли для PostgreSQL (Vault генерирует и ротирует сам)
vault secrets enable database
vault write database/config/mydb \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@postgres:5432/mydb" \
    allowed_roles="app-role" \
    username="vault-admin" \
    password="vault-password"

vault write database/roles/app-role \
    db_name=mydb \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT app_role TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"

# Приложение получает временный пароль:
vault read database/creds/app-role
# Key       Value
# username  v-myapp-xK2m3
# password  A1B2C3...
# lease_duration 1h
```

---

## Чеклист при создании нового сервиса

```
□ Отдельный пользователь БД (не admin/root)
□ Права — только то что нужно (CRUD без DROP/TRUNCATE)
□ Ограничение по IP/сети (pg_hba, bindIp, iptables)
□ Пароль — минимум 20 символов, случайный (pwgen/openssl)
□ Отдельный ServiceAccount в K8s (не default)
□ automountServiceAccountToken: false если токен не нужен
□ Системный Linux-пользователь без shell (nologin)
□ Логи аудита включены
□ Пароль занесён в Vault / секрет-менеджер
□ Ротация пароля — 90 дней
```

---

## Официальные источники

- PostgreSQL RBAC: https://www.postgresql.org/docs/current/user-manag.html
- PostgreSQL RLS: https://www.postgresql.org/docs/current/ddl-rowsecurity.html
- MongoDB RBAC: https://www.mongodb.com/docs/manual/core/authorization/
- ClickHouse Access Control: https://clickhouse.com/docs/en/operations/access-rights
- Kubernetes RBAC: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- Dex OIDC: https://dexidp.io/docs/
- SSSD: https://sssd.io/docs/
- HashiCorp Vault: https://developer.hashicorp.com/vault/docs/secrets/databases

---

## Подводные камни

1. **Default ServiceAccount в K8s** — у него по умолчанию нет прав, но некоторые чарты монтируют его токен. Всегда создавать отдельный SA с `automountServiceAccountToken: false`.

2. **PostgreSQL `public` schema** — по умолчанию все пользователи могут создавать объекты в `public`. Отозвать: `REVOKE CREATE ON SCHEMA public FROM PUBLIC;`.

3. **MongoDB без авторизации** — по умолчанию `authorization: disabled`. Если MongoDB доступна из сети без авторизации — это критическая уязвимость (CVE известны массовые взломы).

4. **`cluster-admin` через LDAP** — если дать группе LDAP cluster-admin и LDAP-сервер компрометирован, атакующий получает полный доступ к кластеру. Для cluster-admin предпочтительнее статические kubeconfig-файлы с отдельными сертификатами.

5. **ClickHouse `default` пользователь** — по умолчанию без пароля. Первым делом установить пароль или отключить: `<password_sha256_hex>...</password_sha256_hex>`.

6. **LDAP bind-пароль** — не хранить в конфигах открытым текстом. Использовать Vault или Kubernetes Secret с RBAC ограничением на чтение.

7. **Rotate, не delete** — при смене пароля сначала создать новый, обновить приложение, затем удалить старый. Иначе даунтайм.
