# Kubernetes Audit Logging — централизованный аудит событий

## Что такое Kubernetes Audit Log

Kubernetes API Server записывает **каждый запрос** к кластеру: кто, что, когда и с каким результатом.
Это единственный полный источник правды о происходящем в кластере.

**Что попадает в аудит:**
- Деплой workloads (Deployment, StatefulSet, DaemonSet, Pod)
- Вызовы Kyverno webhook (срабатывание политик — mutate/validate/generate)
- Изменения RBAC (Role, ClusterRole, RoleBinding)
- Создание/удаление Secret, ConfigMap
- Exec в поды (`kubectl exec`)
- Port-forward сессии
- Изменения NetworkPolicy, Service, Ingress
- Любые API-вызовы (включая CRD)

---

## Настройка Audit Policy

Audit Policy определяет **что** и **с каким уровнем детализации** логировать.

### Уровни аудита

| Уровень | Что пишется |
|---------|-------------|
| `None` | Ничего |
| `Metadata` | Только метаданные (кто, что, когда, HTTP статус) |
| `Request` | Метаданные + тело запроса |
| `RequestResponse` | Метаданные + тело запроса + тело ответа |

### Пример Audit Policy

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Не логировать служебный шум
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
      - group: ""
        resources: ["endpoints", "services", "services/status"]

  - level: None
    userGroups: ["system:nodes"]
    verbs: ["get"]
    resources:
      - group: ""
        resources: ["nodes", "nodes/status"]

  - level: None
    users:
      - system:apiserver
      - system:kube-controller-manager
      - system:kube-scheduler
    verbs: ["get", "list", "watch"]

  # Не логировать readonly запросы к healthz/livez/readyz
  - level: None
    nonResourceURLs:
      - /healthz*
      - /livez*
      - /readyz*
      - /version
      - /metrics

  # Kyverno webhook calls — логировать на уровне Request
  - level: Request
    users: ["system:serviceaccount:kyverno:kyverno-admission-controller"]

  # Секреты — только метаданные (не пишем содержимое в лог!)
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]

  # exec / attach / port-forward — полный лог
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach", "pods/portforward"]

  # RBAC изменения — полный лог
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
        resources:
          ["roles", "clusterroles", "rolebindings", "clusterrolebindings"]

  # Деплои — полный лог
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: "apps"
        resources:
          ["deployments", "statefulsets", "daemonsets", "replicasets"]
      - group: ""
        resources: ["pods"]

  # Всё остальное — только метаданные
  - level: Metadata
```

### Включить на API Server (kubeadm)

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    - --audit-log-maxage=30          # хранить 30 дней
    - --audit-log-maxbackup=10       # макс. 10 файлов
    - --audit-log-maxsize=100        # макс. 100 MB на файл
    # Или вебхук напрямую в коллектор:
    # --audit-webhook-config-file=/etc/kubernetes/audit-webhook.yaml
    # --audit-webhook-batch-max-size=400
    # --audit-webhook-batch-throttle-qps=10
    volumeMounts:
    - mountPath: /etc/kubernetes/audit-policy.yaml
      name: audit-policy
      readOnly: true
    - mountPath: /var/log/kubernetes/audit
      name: audit-log
  volumes:
  - name: audit-policy
    hostPath:
      path: /etc/kubernetes/audit-policy.yaml
      type: File
  - name: audit-log
    hostPath:
      path: /var/log/kubernetes/audit
      type: DirectoryOrCreate
```

### Managed кластера (EKS / GKE / AKS)

```bash
# EKS — включить аудит через ClusterLogging
aws eks update-cluster-config --name my-cluster \
  --logging '{"clusterLogging":[{"types":["audit"],"enabled":true}]}'

# GKE — включить через настройки кластера (по умолчанию включён)
# Логи в Cloud Logging (бывший Stackdriver)

# AKS — включить через DiagnosticSettings в Azure Portal / az CLI
az monitor diagnostic-settings create \
  --resource <AKS_RESOURCE_ID> \
  --name aks-audit \
  --logs '[{"category":"kube-audit","enabled":true}]' \
  --workspace <LOG_ANALYTICS_WORKSPACE_ID>
```

---

## Куда собирать аудит-логи

### Схема

```
kube-apiserver
      │
      ├─ файл → Fluent Bit / Filebeat
      │                  │
      │                  ▼
      │          OpenSearch / Elasticsearch / Loki
      │                  │
      │           Kibana / Grafana ──► AlertManager ──► Telegram/Slack
      │
      └─ webhook → Falco (real-time детекция)
                        │
                   Falcosidekick ──► Telegram / Slack / webhook
```

---

## Инструменты

### 1. Falco — real-time детекция угроз (лучший для security)

Анализирует аудит-логи И системные вызовы ядра в реальном времени.
Имеет готовые правила для подозрительной активности.

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set falco.grpc.enabled=true \
  --set falco.grpcOutput.enabled=true \
  -f falco-values.yaml
```

**falco-values.yaml:**
```yaml
falco:
  rules_file:
    - /etc/falco/falco_rules.yaml
    - /etc/falco/k8s_audit_rules.yaml   # правила аудита K8s
    - /etc/falco/rules.d

  # Подключить K8s Audit через webhook
  k8s_audit_rules:
    enabled: true

  webserver:
    enabled: true
    listen_port: 8765

falcosidekick:
  enabled: true
  config:
    telegram:
      token: "<BOT_TOKEN>"
      chatid: "<CHAT_ID>"
    slack:
      webhookurl: "https://hooks.slack.com/services/..."
    webhook:
      address: "https://alertmanager.internal/api/v2/alerts"
```

**Встроенные правила Falco (примеры):**
```yaml
# Обнаружить shell в поде
- rule: Terminal shell in container
  desc: A shell was used as the entrypoint/exec point into a container
  condition: spawned_process and container and shell_procs and proc.tty != 0
  output: "Shell opened in container (user=%user.name container=%container.name image=%container.image)"
  priority: WARNING

# Запись в /etc внутри контейнера
- rule: Write below etc
  condition: open_write and container and fd.name startswith /etc
  priority: ERROR

# kubectl exec
- rule: K8s Exec into Pod
  condition: k8s_audit and ka.verb=create and ka.target.subresource=exec
  output: "Exec into pod (user=%ka.user.name pod=%ka.target.name ns=%ka.target.namespace)"
  priority: WARNING
```

**Подключить K8s Audit webhook к Falco:**
```yaml
# /etc/kubernetes/audit-webhook.yaml
apiVersion: v1
kind: Config
clusters:
  - name: falco
    cluster:
      server: http://falco.falco.svc:8765/k8s-audit
contexts:
  - name: falco
    context:
      cluster: falco
      user: ""
current-context: falco
```

---

### 2. OpenSearch + Fluent Bit — хранение и поиск

Лучший выбор для compliance: хранит логи, позволяет строить дашборды, делать поиск по истории.

**Fluent Bit DaemonSet для сбора аудит-логов:**
```yaml
# ConfigMap для Fluent Bit
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [INPUT]
        Name              tail
        Tag               k8s.audit
        Path              /var/log/kubernetes/audit/audit.log
        Parser            json
        DB                /var/log/flb_audit.db
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    [FILTER]
        Name    modify
        Match   k8s.audit
        Add     cluster production

    [OUTPUT]
        Name            opensearch
        Match           k8s.audit
        Host            opensearch.logging.svc
        Port            9200
        Index           k8s-audit
        Type            _doc
        Logstash_Format On
        Logstash_Prefix k8s-audit
        tls             Off
```

**Helm установка OpenSearch + Dashboards:**
```bash
helm repo add opensearch https://opensearch-project.github.io/helm-charts
helm install opensearch opensearch/opensearch \
  --namespace logging --create-namespace \
  --set replicas=1 \
  --set persistence.size=50Gi

helm install opensearch-dashboards opensearch/opensearch-dashboards \
  --namespace logging
```

**Полезные запросы в OpenSearch Dashboards:**
```json
// Кто делал exec в поды за последние 24 часа
{
  "query": {
    "bool": {
      "must": [
        { "match": { "verb": "create" } },
        { "match": { "requestURI": "*exec*" } }
      ],
      "filter": {
        "range": { "@timestamp": { "gte": "now-24h" } }
      }
    }
  }
}

// Все деплои в production namespace
{
  "query": {
    "bool": {
      "must": [
        { "match": { "objectRef.namespace": "production" } },
        { "match": { "objectRef.resource": "deployments" } },
        { "terms": { "verb": ["create", "update", "patch"] } }
      ]
    }
  }
}
```

---

### 3. Loki + Grafana — если Grafana уже есть

Если в стеке уже есть Grafana, проще добавить Loki чем поднимать OpenSearch.

```yaml
# Fluent Bit output для Loki
[OUTPUT]
    Name            loki
    Match           k8s.audit
    Host            loki.monitoring.svc
    Port            3100
    Labels          job=k8s-audit,cluster=production
    Line_Format     json
```

**LogQL запросы в Grafana:**
```logql
# Все exec в поды
{job="k8s-audit"} |= "exec" | json | verb="create"

# Изменения секретов
{job="k8s-audit"} | json
  | objectRef_resource="secrets"
  | verb=~"create|update|patch|delete"

# Действия конкретного пользователя
{job="k8s-audit"} | json | user_username="johndoe@company.com"
```

---

### 4. Wazuh — полноценный open source SIEM

Если нужен единый SIEM для всей инфраструктуры (не только K8s):

```bash
# Установка через Docker Compose
git clone https://github.com/wazuh/wazuh-docker
cd wazuh-docker/single-node
docker compose up -d
```

Wazuh имеет готовую интеграцию с Kubernetes:
- Ingests audit logs через Filebeat/Wazuh agent
- Корреляция событий между K8s и хостовой системой
- Compliance dashboards (PCI DSS, HIPAA, GDPR, CIS)
- Встроенные алерты на аномалии

---

## Сравнение инструментов

| | Falco | OpenSearch | Loki + Grafana | Wazuh |
|---|---|---|---|---|
| Real-time алерты | ✅ | через AlertManager | через AlertManager | ✅ |
| Хранение истории | ❌ | ✅ | ✅ | ✅ |
| Поиск по логам | ❌ | ✅ полный | ✅ | ✅ |
| Готовые K8s правила | ✅ | ❌ (писать самому) | ❌ | ✅ |
| System calls | ✅ | ❌ | ❌ | ✅ |
| Ресурсы | низкие | высокие | средние | высокие |
| Сложность | средняя | высокая | низкая (если есть Grafana) | высокая |

**Рекомендации:**
- **Security-фокус, нужны алерты** → Falco + Falcosidekick
- **Compliance, нужна история + поиск** → OpenSearch + Fluent Bit
- **Grafana уже есть** → Loki + Grafana (минимальные усилия)
- **Единый SIEM для всей инфры** → Wazuh
- **Максимальное покрытие** → Falco (real-time) + Loki/OpenSearch (история)

---

## Официальные источники

- Kubernetes Audit: https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/
- Falco: https://falco.org/docs/ | https://github.com/falcosecurity/falco
- Falcosidekick: https://github.com/falcosecurity/falcosidekick
- OpenSearch: https://opensearch.org/docs/
- Wazuh K8s: https://documentation.wazuh.com/current/container-security/kubernetes.html

---

## Подводные камни

1. **Аудит-логи огромные** — на активном кластере могут генерировать гигабайты в день. Обязательно настраивать `--audit-log-maxsize`, `--audit-log-maxbackup` и политику с `None` для служебного шума.

2. **Secrets в Request/RequestResponse уровне** — содержимое Secret попадёт в лог в base64. Для секретов использовать только `Metadata`.

3. **Falco и managed кластера** — в EKS/GKE/AKS нет доступа к kube-apiserver, поэтому audit webhook к Falco не подключить. Используй cloud-native audit (CloudWatch / Cloud Logging / Azure Monitor) → export в OpenSearch/Loki.

4. **Fluent Bit на control-plane нодах** — `/var/log/kubernetes/audit/audit.log` находится на master-нодах. DaemonSet нужно деплоить с tolerations для master:
   ```yaml
   tolerations:
     - key: node-role.kubernetes.io/control-plane
       operator: Exists
       effect: NoSchedule
   ```

5. **Retention** — для compliance обычно требуется хранить аудит-логи 1 год. Планируй хранилище заранее.

6. **Kyverno webhook видно в аудите** — все вызовы admission webhook пишутся в аудит с `user=system:serviceaccount:kyverno:...`. Можно фильтровать чтобы не спамить лог, или наоборот отслеживать каждое срабатывание политики.
