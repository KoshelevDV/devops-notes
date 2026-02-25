# Kyverno — Policy Engine для Kubernetes

## Проблема

Нужно:
- Автоматически подменять образы в workloads (например, `bitnami` → `bitnamilegacy` или внутренний registry)
- Запрещать деплой образов без тегов (`:latest`)
- Добавлять обязательные лейблы/аннотации ко всем ресурсам
- Задавать лимиты ресурсов по умолчанию
- Аудировать кластер на соответствие политикам

## Решение — Kyverno

Kyverno — Kubernetes-нативный policy engine. Работает как MutatingWebhookConfiguration и ValidatingWebhookConfiguration.
Три типа политик: **mutate** (мутировать), **validate** (запрещать), **generate** (генерировать ресурсы).

---

## Установка

```bash
# Helm (рекомендуется)
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace

# Проверить
kubectl get pods -n kyverno
```

### Режимы webhook

По умолчанию Kyverno устанавливает webhook в режиме `Fail` — если Kyverno недоступен, поды не создаются.
Для нагруженных кластеров или первичной установки:

```bash
helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace \
  --set webhooksCleanupEnabled=true \
  --set "webhookFailurePolicy=Ignore"  # мягкий старт
```

---

## Mutate — подмена образов

### Заменить registry целиком

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: replace-image-registry
spec:
  rules:
    - name: replace-registry
      match:
        any:
          - resources:
              kinds: ["Pod"]
      mutate:
        foreach:
          - list: "request.object.spec.containers"
            patchStrategicMerge:
              spec:
                containers:
                  - name: "{{ element.name }}"
                    image: >-
                      {{ regex_replace_all(
                           '^docker.io/',
                           element.image,
                           'registry.internal.company.com/'
                         )
                      }}
```

### Заменить только bitnami → bitnamilegacy

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: replace-bitnami-images
spec:
  rules:
    - name: rewrite-bitnami
      match:
        any:
          - resources:
              kinds: ["Pod"]
      mutate:
        foreach:
          - list: "request.object.spec.containers"
            patchStrategicMerge:
              spec:
                containers:
                  - name: "{{ element.name }}"
                    image: >-
                      {{ regex_replace_all(
                           'docker.io/bitnami/',
                           element.image,
                           'docker.io/bitnamilegacy/'
                         )
                      }}
```

### Покрыть initContainers тоже

```yaml
mutate:
  foreach:
    - list: "request.object.spec.containers"
      patchStrategicMerge:
        spec:
          containers:
            - name: "{{ element.name }}"
              image: "{{ regex_replace_all('bitnami/', element.image, 'bitnamilegacy/') }}"
    - list: "request.object.spec.initContainers"
      patchStrategicMerge:
        spec:
          initContainers:
            - name: "{{ element.name }}"
              image: "{{ regex_replace_all('bitnami/', element.image, 'bitnamilegacy/') }}"
```

### Ограничить по namespace

```yaml
match:
  any:
    - resources:
        kinds: ["Pod"]
        namespaces: ["production", "staging"]
```

### Исключить namespace kyverno и kube-system

```yaml
exclude:
  any:
    - resources:
        namespaces: ["kube-system", "kyverno"]
```

---

## Validate — запрет небезопасных практик

### Запретить latest-теги

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce   # или Audit (только логирование)
  rules:
    - name: no-latest
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Тег ':latest' запрещён. Укажи конкретную версию."
        foreach:
          - list: "request.object.spec.containers"
            deny:
              conditions:
                any:
                  - key: "{{ element.image }}"
                    operator: Equals
                    value: "*:latest"
                  - key: "{{ element.image }}"
                    operator: NotContains
                    value: ":"
```

### Обязательные лейблы

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-labels
      match:
        any:
          - resources:
              kinds: ["Deployment", "StatefulSet", "DaemonSet"]
      validate:
        message: "Обязательные лейблы: app.kubernetes.io/name, app.kubernetes.io/version"
        pattern:
          metadata:
            labels:
              app.kubernetes.io/name: "?*"
              app.kubernetes.io/version: "?*"
```

### Запрет privileged контейнеров

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-privileged
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Privileged контейнеры запрещены"
        pattern:
          spec:
            containers:
              - =(securityContext):
                  =(privileged): "false"
```

---

## Generate — автоматическая генерация ресурсов

### Копировать Secret в новые namespace

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: sync-registry-secret
spec:
  rules:
    - name: copy-imagepullsecret
      match:
        any:
          - resources:
              kinds: ["Namespace"]
      generate:
        apiVersion: v1
        kind: Secret
        name: registry-credentials
        namespace: "{{ request.object.metadata.name }}"
        synchronize: true   # обновлять при изменении источника
        clone:
          namespace: default
          name: registry-credentials
```

---

## Проверка и отладка

```bash
# Статус политик
kubectl get clusterpolicy
kubectl get clusterpolicy replace-bitnami-images -o yaml

# PolicyReport — результаты аудита
kubectl get policyreport -A
kubectl get clusterpolicyreport

# Проверить что реально подставится (dry-run)
kubectl run test --image=docker.io/bitnami/postgresql:14 --dry-run=server -o yaml \
  | grep image

# Логи Kyverno
kubectl logs -n kyverno -l app.kubernetes.io/component=kyverno-admission-controller -f

# CLI-проверка политики без кластера
kyverno apply ./policy.yaml --resource ./pod.yaml
```

### Установить Kyverno CLI

```bash
# macOS
brew install kyverno

# Linux
curl -LO https://github.com/kyverno/kyverno/releases/latest/download/kyverno-cli_linux_x86_64.tar.gz
tar xf kyverno-cli_linux_x86_64.tar.gz
sudo mv kyverno /usr/local/bin/
```

---

## validationFailureAction

| Значение | Поведение |
|----------|-----------|
| `Audit` | Нарушения логируются в PolicyReport, ресурс создаётся |
| `Enforce` | Нарушения блокируют создание ресурса (HTTP 403) |

**Рекомендация:** начинать с `Audit`, смотреть отчёты, потом переключать на `Enforce`.

---

## Официальные источники

- Документация: https://kyverno.io/docs/
- Готовые политики (Kyverno Policy Library): https://kyverno.io/policies/
- GitHub: https://github.com/kyverno/kyverno
- JMESPath + функции Kyverno: https://kyverno.io/docs/writing-policies/jmespath/

---

## Подводные камни

1. **`webhookFailurePolicy: Fail` (по умолчанию)** — если Kyverno pod упал, поды во всём кластере не создаются. На продакшне: настроить PodDisruptionBudget и минимум 3 реплики Kyverno.

2. **foreach и initContainers** — по умолчанию `foreach` по `spec.containers` не покрывает `initContainers` и `ephemeralContainers`. Нужно добавлять отдельные списки.

3. **regex_replace_all** — работает только если образ уже содержит `docker.io/` prefix. Образ `bitnami/postgresql` (без домена) не матчится на `docker.io/bitnami/`. Нужно учитывать оба варианта или нормализовать.

4. **Mutate применяется к Pod, не к Deployment** — Deployment создаёт ReplicaSet, тот создаёт Pod. Webhook срабатывает при создании Pod. В `kubectl describe deployment` будет показан оригинальный образ, но реальные поды будут с подменённым.

5. **Порядок foreach** — в одном rule нельзя иметь два `foreach` для разных списков до Kyverno 1.10. Обновляйтесь.

6. **Namespace kyverno нужно исключать** — иначе Kyverno может попытаться применить политику к собственным подам и зациклиться.
