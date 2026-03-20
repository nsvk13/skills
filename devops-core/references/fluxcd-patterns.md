# FluxCD Patterns Reference

## Bootstrap (GitLab)
```bash
flux bootstrap gitlab \
  --owner=<group> \
  --repository=<repo> \
  --branch=main \
  --path=clusters/prod \
  --token-auth
```

## ImageUpdateAutomation
```yaml
# 1. ImageRepository — следим за registry
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: app
  namespace: flux-system
spec:
  image: registry.gitlab.com/org/app
  interval: 1m
  secretRef:
    name: registry-credentials
---
# 2. ImagePolicy — правило выбора тега
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: app
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: app
  policy:
    semver:
      range: ">=1.0.0"
---
# 3. В манифесте — маркер для автообновления
image: registry.gitlab.com/org/app:1.0.0 # {"$imagepolicy": "flux-system:app"}
---
# 4. ImageUpdateAutomation — пишет коммит в git
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: fleet-repo
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxbot@example.com
        name: FluxBot
    push:
      branch: main
  update:
    path: ./clusters/prod
    strategy: Setters
```

## Multi-tenant структура
```
clusters/
└── prod/
    ├── flux-system/        # bootstrap компоненты
    ├── infrastructure/     # cert-manager, ingress, monitoring
    │   ├── kustomization.yaml
    │   └── ...
    └── apps/               # прикладные сервисы
        ├── kustomization.yaml
        └── ...
```

## Отладка
```bash
# Посмотреть что не синхронизировано
flux get all -A | grep -v True

# Принудительная синхронизация
flux reconcile ks flux-system --with-source

# Логи контроллера
flux logs --kind=HelmRelease --name=app -n flux-system

# Suspend/resume (для ручного деплоя)
flux suspend hr app -n flux-system
flux resume hr app -n flux-system
```

## HelmRelease — values из Secret
```yaml
spec:
  valuesFrom:
    - kind: Secret
      name: app-values
      valuesKey: values.yaml
```