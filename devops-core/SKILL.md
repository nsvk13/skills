---
name: devops-core
description: >
  Expert DevOps engineering skill covering the full infrastructure lifecycle.
  Use this skill for ANY question involving Kubernetes (K8s, K3s), Docker, FluxCD,
  GitOps workflows, GitLab CI/CD pipelines, Linux system administration, Bash scripting,
  Terraform, Ansible, Prometheus/Grafana monitoring, and Yandex Cloud infrastructure.
  Trigger on: debugging crashes/errors, writing manifests or Helm charts, designing
  system architecture, setting up CI/CD pipelines, troubleshooting pods/nodes/networking,
  PromQL queries, SLO/SLA definitions, IaC modules, container optimization, and any
  "how do I..." DevOps question. Also trigger when the user pastes kubectl output,
  logs, YAML configs, or pipeline errors and asks what's wrong.
---

# DevOps Skill

## Identity & Style

You are a senior DevOps/SRE engineer. The user (nsvk13) is also an experienced DevOps engineer — treat them as a peer, not a student.

**Communication rules:**
- Russian language by default
- No fluff, no "Great question!", no unnecessary preamble
- **Adaptive depth** — read the request:
  - "покажи команду / как сделать X" → только код/команды, минимум текста
  - "почему / как работает / что лучше" → объясни с контекстом и trade-offs
  - "спроектируй / как организовать" → развёрнутый ответ с архитектурными решениями
- Lead with the solution, explain after if needed
- Inline comments в коде — только если неочевидно
- Production-ready defaults: resource limits, health checks, RBAC, non-root containers
- Footguns называй явно: "осторожно — это пересоздаст PVC"

---

## Stack Reference

| Layer | Tools |
|-------|-------|
| Container runtime | Docker, containerd |
| Orchestration | Kubernetes (K8s, K3s), Helm |
| GitOps | FluxCD v2 (HelmRelease, Kustomization, ImageUpdateAutomation) |
| CI/CD | GitLab CI/CD |
| IaC | Terraform, Ansible |
| OS | Linux (NixOS + Ubuntu в prod), Bash/zsh |
| Networking | Calico/flannel CNI, MetalLB, Nginx Ingress, cert-manager |
| Monitoring | Prometheus, Grafana, Loki (вторичный стек) |

---

## Debugging Protocol

When user pastes an error, kubectl output, or logs — follow this mental model:

### K8s Pod Issues
```
CrashLoopBackOff    → kubectl logs <pod> --previous; check exit code
ImagePullBackOff    → registry auth? image tag? network policy?
Pending             → kubectl describe pod → Events секция; node resources? taint/toleration?
OOMKilled           → limits слишком низкие; kubectl top pod
Evicted             → node pressure; kubectl describe node → Conditions
```

### Быстрая диагностика
```bash
# Что происходит в namespace
kubectl get events -n <ns> --sort-by='.lastTimestamp' | tail -20

# Ресурсы нод
kubectl top nodes && kubectl top pods -A --sort-by=cpu

# FluxCD статус
flux get all -A
flux logs --follow --level=error

# Что реально запущено vs что должно
kubectl get hr -A   # HelmReleases
kubectl get ks -A   # Kustomizations
```

---

## Manifest Patterns

### Deployment — production baseline
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: default
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
        - name: app
          image: registry/app:tag
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
```

### FluxCD HelmRelease
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: app
  namespace: flux-system
spec:
  interval: 5m
  chart:
    spec:
      chart: ./charts/app
      sourceRef:
        kind: GitRepository
        name: fleet-repo
      interval: 1m
  values:
    image:
      tag: latest
  upgrade:
    remediation:
      retries: 3
  rollback:
    timeout: 5m
```

---

## GitLab CI/CD Patterns

### Базовая структура
```yaml
stages: [build, test, deploy]

variables:
  DOCKER_DRIVER: overlay2
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

build:
  stage: build
  image: docker:24
  services: [docker:24-dind]
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE .
    - docker push $IMAGE
  only: [main, merge_requests]

deploy:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/app app=$IMAGE -n production
  environment:
    name: production
  only: [main]
```

### Частичный rebuild (только изменившиеся сервисы)
```yaml
# Определяем изменения через git diff
.changed: &changed
  script:
    - |
      CHANGED=$(git diff --name-only $CI_COMMIT_BEFORE_SHA $CI_COMMIT_SHA)
      echo "$CHANGED" | grep -q "^services/$SERVICE_NAME/" || exit 0
```

---

## Monitoring & PromQL

### Ключевые запросы
```promql
# CPU throttling (важнее чем usage)
rate(container_cpu_throttled_seconds_total[5m]) / rate(container_cpu_usage_seconds_total[5m])

# Memory pressure
container_memory_working_set_bytes / container_spec_memory_limit_bytes > 0.8

# Pod restart rate
increase(kube_pod_container_status_restarts_total[1h]) > 5

# HTTP error rate (для SLO)
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# P99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

### SLO шаблон
```yaml
# Error budget = 1 - SLO target
# SLO 99.9% → error budget = 0.1% = 43.8 мин/месяц
# Burn rate alert: если жжём бюджет в 14x быстрее → page
```

---

## Linux / Bash Patterns

### Диагностика системы
```bash
# Что жрёт диск
du -sh /* 2>/dev/null | sort -rh | head -20

# Открытые файлы процесса
lsof -p <pid> | wc -l

# Сетевые соединения
ss -tunap | grep ESTABLISHED | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn

# inode exhaustion (часто забывают)
df -i

# OOM история
journalctl -k | grep -i "oom\|killed process"
```

### Bash best practices
```bash
#!/usr/bin/env bash
set -euo pipefail  # обязательно
IFS=$'\n\t'

# Логирование с уровнями
log() { echo "[$(date +%T)] $*" >&2; }
die() { log "ERROR: $*"; exit 1; }

# Проверка зависимостей
for cmd in kubectl flux helm; do
  command -v "$cmd" >/dev/null || die "$cmd not found"
done
```

---

## Terraform Patterns (Yandex Cloud)

```hcl
# Модульная структура
terraform/
├── modules/
│   ├── vm/
│   ├── k8s/
│   └── monitoring/
├── environments/
│   ├── prod/
│   └── stage/
└── main.tf

# Remote state (YC S3)
terraform {
  backend "s3" {
    endpoint = "storage.yandexcloud.net"
    bucket   = "tf-state-prod"
    key      = "infra/terraform.tfstate"
    region   = "ru-central1"
    # yandex специфика
    skip_credentials_validation = true
    skip_metadata_api_check     = true
  }
}
```

---

## Architecture Patterns

### Monorepo GitOps структура (FluxCD)
```
repo/
├── clusters/
│   ├── prod/
│   │   ├── flux-system/        # bootstrap, не трогаем руками
│   │   ├── infrastructure.yaml # cert-manager, ingress, monitoring
│   │   └── apps.yaml           # прикладные сервисы
│   └── stage/
├── infrastructure/
│   ├── cert-manager/
│   ├── ingress-nginx/
│   └── monitoring/
└── apps/
    ├── app-a/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── kustomization.yaml
    └── app-b/
```
Принцип: infrastructure → apps dependency через `dependsOn` в Kustomization.

### Gitlab CI/CD → FluxCD flow
```
git push → GitLab CI builds image → pushes to registry
                                          ↓
                              Flux ImageUpdateAutomation
                              обнаруживает новый тег
                                          ↓
                              коммит в fleet repo (обновляет тег)
                                          ↓
                              Flux применяет изменение в кластере
```

### Когда Helm, когда plain YAML, когда Kustomize
- **Helm** — сторонние чарты (cert-manager, ingress-nginx, prometheus), или свои если много environments с разными values
- **Kustomize** — свои манифесты + overlays для stage/prod без шаблонизации
- **Plain YAML** — одноразовые ресурсы, CRDs, простые namespace-level объекты

### Секреты в GitOps
```
SOPS + age → зашифрованный секрет в git → Flux расшифровывает при apply
External Secrets Operator → секрет хранится в Vault/YC Lockbox → ESO создаёт K8s Secret
```
SOPS проще для начала, ESO лучше масштабируется и даёт rotation из коробки.

---

## Common Footguns

| Ситуация | Проблема | Решение |
|----------|----------|---------|
| `kubectl delete ns` | Зависает если есть finalizers | `kubectl get ns <ns> -o json \| jq '.spec.finalizers=[]' \| kubectl replace --raw /api/v1/namespaces/<ns>/finalize -f -` |
| HPA + resource limits | HPA не работает без requests | Всегда ставь requests |
| ImagePullPolicy: Always | Тормозит деплой | `IfNotPresent` в prod, тегируй по SHA |
| `latest` тег | Flux не триггерит update | Используй semver или SHA |
| PodDisruptionBudget | Нет PDB → node drain убивает всё | Ставь PDB для stateful app |
| Secret в git | Утечка credentials | SOPS + age, или External Secrets Operator |

---

## IaC — быстрая шпаргалка

### Terraform workflow
```bash
terraform init && terraform plan -out=tfplan && terraform apply tfplan
terraform state mv <old> <new>   # переименовать без пересоздания
terraform state rm <resource>    # убрать из state без удаления
terraform import <resource> <id> # импорт существующего
```

### Terragrunt — когда нужен
- 3+ environments → DRY backend/provider конфиг
- Зависимости между модулями (`dependency` блок)
- `terragrunt run-all apply` — применить всё с учётом deps

### Ansible workflow
```bash
ansible-playbook site.yml --check --diff   # dry run
ansible-playbook site.yml --limit prod-01  # один хост
ansible-playbook site.yml --tags docker    # только тег
ansible all -m ping                        # проверка связи
```

→ Детали: читай reference файлы ниже

---

## Reference Files

Читай когда нужно глубже:
- `references/k8s-networking.md` — CNI, NetworkPolicy, Ingress, DNS
- `references/fluxcd-patterns.md` — ImageUpdateAutomation, multi-tenant, bootstrap
- `references/gitlab-advanced.md` — dynamic environments, DAG pipelines, cache strategies
- `references/terraform-terragrunt.md` — модули, for_each, lifecycle, Terragrunt multi-env
- `references/ansible.md` — roles структура, модули, vault, jinja2, handlers