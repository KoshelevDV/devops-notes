# FluxCD — развёртывание infra layer в Kubernetes

> GitOps-подход для управления инфраструктурным слоем k8s: Helm charts, манифесты, зависимости, секреты, мониторинг.
> Версия документации: Flux v2 (latest stable). Источник: https://fluxcd.io/flux/

---

## Компоненты Flux

| Контроллер | Назначение |
|---|---|
| **source-controller** | Следит за Git репо, HelmRepository, OCI Registry |
| **kustomize-controller** | Применяет Kustomize overlays и plain YAML-манифесты |
| **helm-controller** | Управляет HelmRelease (install/upgrade/rollback/drift detection) |
| **notification-controller** | Алерты, webhook receivers |
| **image-automation-controller** | Автообновление image tags (опционально) |

---

## Установка и bootstrap

### Установка CLI

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
# Completions
. <(flux completion bash)
```

### Bootstrap с GitLab

```bash
export GITLAB_TOKEN=<gl-pat>

# Self-hosted GitLab
flux bootstrap gitlab \
  --token-auth \
  --hostname=gitlab.example.com \
  --owner=my-group \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/prod

# GitLab с deploy token (рекомендуется для CI/CD)
flux bootstrap gitlab \
  --deploy-token-auth \
  --owner=my-group/my-subgroup \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/prod
```

**PAT требует:** full read/write access to GitLab API.
Bootstrap идемпотентен — можно запускать повторно для upgrade Flux.

### Bootstrap с GitHub

```bash
export GITHUB_TOKEN=<gh-token>

flux bootstrap github \
  --owner=my-org \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/prod \
  --personal
```

---

## Структура репозитория

```
fleet-infra/
├── clusters/
│   └── prod/
│       ├── flux-system/          # Flux controllers (auto-generated)
│       └── infra.yaml            # Kustomization entry point
├── infrastructure/
│   ├── namespaces/               # Namespaces
│   ├── sources/                  # HelmRepository, GitRepository
│   ├── controllers/              # metallb, nginx-ingress, cert-manager
│   └── apps/                     # kafka, redis, postgresql
└── apps/
    └── prod/                     # Application Helm releases
```

---

## Sources — откуда брать чарты

### HTTP/S Helm репозиторий (публичный)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: ingress-nginx
  namespace: flux-system
spec:
  interval: 1h
  url: https://kubernetes.github.io/ingress-nginx
```

### HTTP/S Helm репозиторий с авторизацией (приватный)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: private-charts
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.example.com
  secretRef:
    name: private-charts-auth
---
apiVersion: v1
kind: Secret
metadata:
  name: private-charts-auth
  namespace: flux-system
stringData:
  username: my-user
  password: my-password
```

### OCI Registry (рекомендуется для современных чартов)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 5m
  url: oci://ghcr.io/stefanprodan/charts/podinfo
  layerSelector:
    mediaType: "application/vnd.cncf.helm.chart.content.v1.tar+gzip"
    operation: copy
  ref:
    semver: "^6.9.0"
```

### OCI Registry с авторизацией (GitLab Registry)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m
  url: oci://registry.gitlab.example.com/my-group/my-charts/my-app
  ref:
    semver: ">=1.0.0"
  secretRef:
    name: gitlab-registry-auth
---
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-registry-auth
  namespace: flux-system
stringData:
  username: deploy-token-user
  password: deploy-token-value
```

> Секрет для OCI/Docker Registry должен быть типа `kubernetes.io/dockerconfigjson`.
> Создать: `kubectl create secret docker-registry gitlab-registry-auth --docker-server=... --docker-username=... --docker-password=...`

### Локальный Helm чарт из Git репозитория

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: fleet-infra
  namespace: flux-system
spec:
  interval: 1m
  url: https://gitlab.example.com/my-group/fleet-infra
  ref:
    branch: main
  secretRef:
    name: fleet-infra-auth   # для приватного репо
---
# HelmRelease ссылается на путь в GitRepository
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: my-app
  namespace: apps
spec:
  interval: 10m
  chart:
    spec:
      chart: ./helm/my-app       # путь внутри репо
      sourceRef:
        kind: GitRepository
        name: fleet-infra
        namespace: flux-system
```

---

## HelmRelease — установка чартов

### Чарт из HelmRepository (публичный)

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  interval: 1h
  chart:
    spec:
      chart: ingress-nginx
      version: "4.x.x"          # semver range
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
        namespace: flux-system
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
  driftDetection:
    mode: enabled
  values:
    controller:
      replicaCount: 2
      service:
        type: LoadBalancer
```

### Чарт из OCIRepository

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: podinfo
  namespace: default
spec:
  interval: 10m
  chartRef:                     # ссылка на существующий OCIRepository
    kind: OCIRepository
    name: podinfo
    namespace: flux-system
  values:
    replicaCount: 2
```

### values из ConfigMap или Secret

```yaml
spec:
  valuesFrom:
    - kind: ConfigMap
      name: my-app-values
      valuesKey: values.yaml    # ключ в ConfigMap (по умолчанию values.yaml)
    - kind: Secret
      name: my-app-secrets
      valuesKey: values.yaml
      optional: true
```

---

## Kustomization — plain манифесты

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: namespaces
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: "./infrastructure/namespaces"
  prune: true                   # удалять ресурсы, пропавшие из Git
  targetNamespace: ""           # применять в namespace, указанном в манифестах
  timeout: 2m
```

---

## Зависимости — dependsOn

`dependsOn` есть у **Kustomization** и **HelmRelease**. Flux ждёт Ready статуса у зависимости перед применением.

### Пример: metallb → nginx-ingress → apps

```yaml
# 1. metallb — без зависимостей
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: metallb
  namespace: flux-system
spec:
  interval: 1h
  path: ./infrastructure/controllers/metallb
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  prune: true

---
# 2. nginx-ingress — ждёт metallb
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  dependsOn:
    - name: metallb
      namespace: flux-system
  # ...

---
# 3. Apps — ждут ingress-nginx
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  dependsOn:
    - name: ingress-nginx
      namespace: flux-system
  # ...
```

### Как избежать каскада зависимостей

**Проблема:** A → B → C → D → ... — при падении A весь стек заморожен.

**Решения:**

1. **Группировать уровни, а не отдельные компоненты:**
   ```
   namespaces (нет зависимостей)
       ↓
   infrastructure-controllers (metallb, cert-manager — параллельно)
       ↓
   infrastructure-configs (ingress, storage-class — один уровень)
       ↓
   apps (параллельно друг другу)
   ```

2. **healthChecks** — Flux ждёт реальной готовности ресурса, не просто apply:
   ```yaml
   spec:
     healthChecks:
       - apiVersion: apps/v1
         kind: Deployment
         name: metallb-controller
         namespace: metallb-system
     healthCheckTimeout: 3m
   ```

3. **Минимум цепочек** — dependsOn только на компоненты, которые реально нужны для старта (не "на всякий случай").

4. **Suspend/Resume** — при проблемах изолируй компонент: `flux suspend kustomization metallb`

---

## Примеры infra layer

### Namespaces

```yaml
# infrastructure/namespaces/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - metallb-system.yaml
  - ingress-nginx.yaml
  - kafka.yaml
  - redis.yaml
  - postgresql.yaml
```

```yaml
# infrastructure/namespaces/metallb-system.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: metallb-system
```

### MetalLB (HelmRelease)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: metallb
  namespace: flux-system
spec:
  interval: 1h
  url: https://metallb.github.io/metallb
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: metallb
  namespace: metallb-system
spec:
  interval: 1h
  dependsOn:
    - name: namespaces
      namespace: flux-system
  chart:
    spec:
      chart: metallb
      version: ">=0.14.0"
      sourceRef:
        kind: HelmRepository
        name: metallb
        namespace: flux-system
  install:
    crds: CreateReplace
  upgrade:
    crds: CreateReplace
  values:
    {}
```

### nginx-ingress (HelmRelease)

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  interval: 1h
  dependsOn:
    - name: metallb
      namespace: flux-system
  chart:
    spec:
      chart: ingress-nginx
      version: "4.x.x"
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
        namespace: flux-system
  values:
    controller:
      replicaCount: 2
```

### Kafka — Strimzi (operator + cluster)

```yaml
# Шаг 1: установить Strimzi operator
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: strimzi
  namespace: flux-system
spec:
  interval: 1h
  url: https://strimzi.io/charts/
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: strimzi-operator
  namespace: kafka
spec:
  interval: 1h
  chart:
    spec:
      chart: strimzi-kafka-operator
      version: ">=0.42.0"
      sourceRef:
        kind: HelmRepository
        name: strimzi
        namespace: flux-system
  install:
    crds: CreateReplace
  upgrade:
    crds: CreateReplace

---
# Шаг 2: Kafka cluster (Kustomization с plain manifest)
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: kafka-cluster
  namespace: flux-system
spec:
  dependsOn:
    - name: strimzi-operator
      namespace: flux-system   # ждём готовности operator
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./infrastructure/apps/kafka
  prune: true
```

### Redis (Bitnami)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.bitnami.com/bitnami
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: redis
  namespace: redis
spec:
  interval: 1h
  chart:
    spec:
      chart: redis
      version: ">=20.0.0"
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
  values:
    architecture: standalone
    auth:
      enabled: true
      existingSecret: redis-auth
      existingSecretPasswordKey: password
```

### CloudNativePG (оператор PostgreSQL)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: cnpg
  namespace: flux-system
spec:
  interval: 1h
  url: https://cloudnative-pg.github.io/charts
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cnpg
  namespace: cnpg-system
spec:
  interval: 1h
  chart:
    spec:
      chart: cloudnative-pg
      version: ">=0.23.0"
      sourceRef:
        kind: HelmRepository
        name: cnpg
        namespace: flux-system
  install:
    crds: CreateReplace
  upgrade:
    crds: CreateReplace
```

---

## Секреты

### SOPS + age (рекомендуется)

```bash
# 1. Генерируем age key
age-keygen -o age.agekey
# Public key: age1helqcqsh9464r8chnwc2fzj8uv7vr5ntnsft0tn45v2xtz0hpfwq98cmsg

# 2. Создаём Secret в кластере
cat age.agekey | kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin

# 3. Шифруем секрет
sops --age=age1helqcqsh9464r8chnwc2fzj8uv7vr5ntnsft0tn45v2xtz0hpfwq98cmsg \
  --encrypt --encrypted-regex '^(data|stringData)$' --in-place my-secret.yaml

# 4. Коммитим зашифрованный файл — в Git хранить безопасно
```

```yaml
# .sops.yaml — конфиг для автоматического шифрования
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    age: age1helqcqsh9464r8chnwc2fzj8uv7vr5ntnsft0tn45v2xtz0hpfwq98cmsg
```

```yaml
# Kustomization с расшифровкой
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-secrets
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./infrastructure/secrets
  prune: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

### Распространение секрета во все namespace

Kubernetes не копирует секреты между namespace нативно. Варианты:

**1. External Secrets Operator (ESO) + ClusterSecretStore** — рекомендуется:
```yaml
# ClusterSecretStore читает из Vault/AWS SM/GitLab CI Variable
apiVersion: external-secrets.io/v1beta1
kind: ClusterExternalSecret
metadata:
  name: registry-auth
spec:
  namespaceSelector:
    matchLabels:
      inject-registry-secret: "true"   # метка на namespace
  externalSecretSpec:
    refreshInterval: 1h
    secretStoreRef:
      name: vault-backend
      kind: ClusterSecretStore
    target:
      name: registry-auth
    data:
      - secretKey: .dockerconfigjson
        remoteRef:
          key: secret/registry
          property: dockerconfig
```

**2. Reflector** (emberstack/kubernetes-reflector) — отражает секрет во все namespace:
```yaml
# На исходном секрете — аннотации
apiVersion: v1
kind: Secret
metadata:
  name: registry-auth
  namespace: flux-system
  annotations:
    reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
    reflector.v1.k8s.emberstack.com/reflection-auto-enabled: "true"
    reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces: ""  # все
```

**3. SOPS Kustomization на каждый namespace** — декларативно, но требует дублирования.

---

## Интеграция с Vault (через External Secrets Operator)

FluxCD не имеет нативной интеграции с Vault. Стандартный подход — ESO.

```yaml
# Установка ESO через Flux
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: external-secrets
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.external-secrets.io
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: external-secrets
  namespace: external-secrets
spec:
  interval: 1h
  chart:
    spec:
      chart: external-secrets
      version: ">=0.10.0"
      sourceRef:
        kind: HelmRepository
        name: external-secrets
        namespace: flux-system
  install:
    crds: CreateReplace
```

```yaml
# ClusterSecretStore → Vault
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "flux-reader"
```

---

## Bootstrap с GitLab CI — автоматический деплой

```yaml
# .gitlab-ci.yml — bootstrap Flux в CD pipeline
deploy:
  stage: deploy
  image: ghcr.io/fluxcd/flux-cli:latest
  script:
    - flux bootstrap gitlab
        --token-auth
        --hostname=$GITLAB_HOST
        --owner=$CI_PROJECT_NAMESPACE
        --repository=fleet-infra
        --branch=main
        --path=clusters/prod
  environment:
    name: production
```

---

## Мониторинг

### Prometheus метрики (порт 8080, /metrics)

Ключевые метрики контроллеров:

```
# Время reconciliation по типу ресурса
gotk_reconcile_duration_seconds_bucket{kind, name, namespace, le}
gotk_reconcile_duration_seconds_sum{kind, name, namespace}

# Состояние ресурсов (через kube-state-metrics)
gotk_resource_info{customresource_kind, exported_namespace, name, ready, suspended}

# Для HelmRelease — дополнительные labels:
# revision, chart_name, chart_app_version, chart_source_name

# CPU/Memory контроллеров
process_cpu_seconds_total{namespace, pod}
container_memory_working_set_bytes{namespace, pod}
```

### Настройка мониторинга

```bash
# Готовый пример от Flux team (kube-prometheus-stack + Loki + Grafana дашборды)
git clone https://github.com/fluxcd/flux2-monitoring-example
```

Включает:
- kube-prometheus-stack
- kube-state-metrics с CustomResourceState для Flux CRDs
- Loki для логов контроллеров
- Готовые Grafana дашборды

### CLI мониторинг

```bash
# Статус всех Flux ресурсов
flux get all -A

# Статус конкретных типов
flux get kustomizations -A
flux get helmreleases -A
flux get sources all -A

# Логи контроллера
flux logs --follow --level=error
flux logs -n flux-system --kind=HelmRelease --name=ingress-nginx

# Принудительная reconciliation
flux reconcile kustomization apps --with-source
flux reconcile helmrelease ingress-nginx -n ingress-nginx

# Приостановить/возобновить
flux suspend helmrelease ingress-nginx -n ingress-nginx
flux resume helmrelease ingress-nginx -n ingress-nginx
```

### Алерты через notification-controller

```yaml
# Отправка алертов в Slack
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: slack-bot
  namespace: flux-system
spec:
  type: slack
  channel: flux-alerts
  secretRef:
    name: slack-url
---
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: on-call-webapp
  namespace: flux-system
spec:
  summary: "Flux reconciliation alert"
  providerRef:
    name: slack-bot
  eventSeverity: error
  eventSources:
    - kind: GitRepository
      name: "*"
    - kind: Kustomization
      name: "*"
    - kind: HelmRelease
      name: "*"
```

---

## Drift Detection и исправление

HelmRelease умеет детектировать и исправлять drift (кто-то вручную изменил ресурс в кластере):

```yaml
spec:
  driftDetection:
    mode: enabled          # enabled | warn | disabled
    ignore:
      - paths: ["/spec/replicas"]    # игнорировать HPA-управляемые поля
        target:
          kind: Deployment
```

---

## Полезные команды

```bash
# Проверить преконфигурацию перед push
flux diff kustomization apps --path ./apps/prod

# Export всех ресурсов
flux export kustomization --all > backup.yaml

# Tree зависимостей
flux tree kustomization flux-system

# Проверить health cluster
flux check

# Статус bootstrap
flux get all -n flux-system
```

---

## Ссылки

- Docs: https://fluxcd.io/flux/
- Bootstrap GitLab: https://fluxcd.io/flux/installation/bootstrap/gitlab/
- HelmRelease API: https://fluxcd.io/flux/components/helm/helmreleases/
- Kustomization API: https://fluxcd.io/flux/components/kustomize/kustomizations/
- SOPS guide: https://fluxcd.io/flux/guides/mozilla-sops/
- Monitoring: https://fluxcd.io/flux/monitoring/metrics/
- Monitoring example: https://github.com/fluxcd/flux2-monitoring-example
- Helm releases guide: https://fluxcd.io/flux/guides/helmreleases/
