# ФСТЭК Приказ №117 — Защита контейнерных сред (ЗКС)

> Источник: https://fstec.ru/dokumenty/vse-dokumenty/spetsialnye-normativnye-dokumenty/trebovaniya-utverzhdeny-prikazom-fstek-rossii-ot-11-aprelya-2025-g-n-117
> Дата публикации: апрель 2025

---

## Раздел 4.5 — Защита технологий контейнерных сред и их оркестрации (ЗКО)

8 мер, которые теперь обязательны для организаций, подпадающих под действие приказа (КИИ, ГИС, ИСПДн соответствующих уровней).

---

## Меры и чем их закрывать

| Мера | Название | Суть | Инструменты |
|---|---|---|---|
| **ЗКС.1** | Контроль целостности | Подписи образов, неизменность запущенных контейнеров | Cosign, Notary v2, admission webhook |
| **ЗКС.2** | Регистрация событий | Аудит API server, runtime-событий (exec, создание контейнеров) | Falco, k8s audit log, Loki |
| **ЗКС.3** | Управление доступом | RBAC, минимальные права ServiceAccount, запрет privileged | OPA Gatekeeper, Kyverno, k8s RBAC |
| **ЗКС.4** | Резервное копирование | Backup Persistent Volumes, etcd snapshots | **Velero**, etcd snapshot cron |
| **ЗКС.5** | Изоляция контейнеров | seccomp, AppArmor, namespace isolation, NetworkPolicy | Pod Security Admission, Cilium, Calico |
| **ЗКС.6** | Идентификация и аутентификация | Аутентификация workloads, mTLS между сервисами | SPIFFE/SPIRE, Istio, cert-manager |
| **ЗКС.7** | Управление образами и оркестрация | Политики деплоя, контроль реестра, запрет latest | Kyverno, Harbor, GitLab Registry |
| **ЗКС.8** | Выявление уязвимостей | Сканирование образов в CI и в кластере | Trivy Operator, Grype, Trivy CLI |

---

## Маппинг на open-source инструменты

### ЗКС.1 — Cosign (подпись образов)

```bash
# Подписать образ
cosign sign --key cosign.key registry.example.com/app:v1.0

# Верификация в admission webhook (Kyverno policy)
# Kyverno проверяет подпись перед деплоем
```

```yaml
# Kyverno ClusterPolicy — запрет непроверенных образов
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  rules:
    - name: verify-signature
      match:
        resources:
          kinds: [Pod]
      verifyImages:
        - imageReferences: ["registry.example.com/*"]
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      ...
                      -----END PUBLIC KEY-----
```

### ЗКС.2 — Falco (runtime security)

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set falco.grpc.enabled=true \
  --set falco.grpcOutput.enabled=true
```

Ключевые правила из коробки:
- `Terminal shell in container` — exec в работающий контейнер
- `Write below etc` — запись в `/etc` внутри контейнера
- `Contact K8S API Server From Container` — обращение к API server из пода

### ЗКС.3 — Kyverno (политики)

```yaml
# Запрет privileged контейнеров
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  rules:
    - name: no-privileged
      match:
        resources:
          kinds: [Pod]
      validate:
        message: "Privileged mode is not allowed"
        pattern:
          spec:
            containers:
              - =(securityContext):
                  =(privileged): "false"
```

### ЗКС.4 — Velero (бэкапы)

```bash
# Установка
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm install velero vmware-tanzu/velero \
  --namespace velero --create-namespace \
  -f values.yaml

# Создать backup
velero backup create daily-backup --include-namespaces production

# Schedule (ежедневно в 02:00)
velero schedule create daily \
  --schedule="0 2 * * *" \
  --include-namespaces production \
  --ttl 720h
```

### ЗКС.5 — Pod Security Admission

```yaml
# Включить restricted профиль для namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### ЗКС.8 — Trivy Operator (сканирование в кластере)

```bash
helm repo add aquasecurity https://aquasecurity.github.io/helm-charts/
helm install trivy-operator aquasecurity/trivy-operator \
  --namespace trivy-system --create-namespace \
  --set trivy.ignoreUnfixed=true
```

Создаёт `VulnerabilityReport` CRDs для каждого workload. Метрики в Prometheus через `trivy-operator-metrics`.

---

## Что уже есть в appsec-platform

| Мера | Статус | Шаблон |
|---|---|---|
| ЗКС.8 | ✅ | `trivy.yml`, `scanall` |
| ЗКС.3 | ⚠️ частично | через CI политики, не Kyverno |
| ЗКС.1 | ❌ | нет Cosign |
| ЗКС.2 | ❌ | нет Falco |
| ЗКС.4 | ❌ | нет Velero |
| ЗКС.5 | ❌ | PSA не настроен |
| ЗКС.6 | ❌ | нет mTLS/SPIFFE |
| ЗКС.7 | ⚠️ частично | Harbor есть, Kyverno нет |

---

## Подводные камни

- **Velero** требует совместимого CSI-драйвера для снапшотов PV; без него бэкап только "crash-consistent"
- **Falco** даёт много шума из коробки — нужна настройка правил под реальный workload
- **Cosign** в GitLab CI требует keyless-режима через OIDC или хранения ключа в Vault
- **Pod Security Admission `restricted`** ломает многие legacy-деплойменты — сначала `warn`, потом `enforce`
- **SPIFFE/SPIRE** — сложная установка; Istio как более простая альтернатива для mTLS

---

## Ссылки

- Приказ №117: https://fstec.ru/dokumenty/vse-dokumenty/spetsialnye-normativnye-dokumenty/trebovaniya-utverzhdeny-prikazom-fstek-rossii-ot-11-aprelya-2025-g-n-117
- Falco: https://falco.org/docs/
- Cosign: https://docs.sigstore.dev/cosign/overview/
- Kyverno: https://kyverno.io/docs/
- Velero: https://velero.io/docs/
- Trivy Operator: https://aquasecurity.github.io/trivy-operator/
