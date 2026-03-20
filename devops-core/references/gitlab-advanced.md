# GitLab CI/CD Advanced Patterns

## DAG Pipeline (needs: вместо stages)
```yaml
build:
  stage: build
  script: [docker build ...]

test-unit:
  needs: [build]
  script: [run tests]

test-integration:
  needs: [build]
  script: [run integration]

deploy:
  needs: [test-unit, test-integration]  # ждёт оба
  script: [kubectl apply ...]
```

## Dynamic Environments
```yaml
deploy-review:
  script:
    - BRANCH=$(echo $CI_COMMIT_REF_SLUG | cut -c1-20)  # лимит длины
    - helm upgrade --install app-$BRANCH ./chart \
        --set image.tag=$CI_COMMIT_SHORT_SHA \
        --namespace review-$BRANCH \
        --create-namespace
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_COMMIT_REF_SLUG.staging.example.com
    on_stop: stop-review
  only: [merge_requests]

stop-review:
  script:
    - helm uninstall app-$CI_COMMIT_REF_SLUG -n review-$CI_COMMIT_REF_SLUG
    - kubectl delete ns review-$CI_COMMIT_REF_SLUG
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  when: manual
```

## Cache стратегия
```yaml
# npm/bun кэш
cache:
  key:
    files: [bun.lockb]
  paths: [node_modules/]
  policy: pull-push

# Docker layer cache через registry
build:
  script:
    - docker build \
        --cache-from $IMAGE:cache \
        --build-arg BUILDKIT_INLINE_CACHE=1 \
        -t $IMAGE:$CI_COMMIT_SHORT_SHA \
        -t $IMAGE:cache .
    - docker push $IMAGE:$CI_COMMIT_SHORT_SHA
    - docker push $IMAGE:cache
```

## Partial rebuild (изменившиеся сервисы)
```yaml
.check-changes: &check-changes |
  CHANGED=$(git diff --name-only ${CI_COMMIT_BEFORE_SHA}..${CI_COMMIT_SHA} 2>/dev/null || git diff --name-only HEAD~1)

build-api:
  script:
    - *check-changes
    - echo "$CHANGED" | grep -q "^services/api/" || { echo "No changes, skipping"; exit 0; }
    - docker build -t $CI_REGISTRY_IMAGE/api:$CI_COMMIT_SHORT_SHA services/api/
```

## Rules — замена only/except
```yaml
deploy-prod:
  rules:
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"'
      when: on_success
    - if: '$FORCE_DEPLOY == "true"'
      when: manual
    - when: never
```

## Branch name нормализация
```yaml
variables:
  SAFE_BRANCH: $(echo $CI_COMMIT_REF_SLUG | tr '/' '-' | cut -c1-20)
```

## No-autodelete защита
```yaml
stop-stand:
  script:
    - |
      LABELS=$(kubectl get ns $NAMESPACE -o jsonpath='{.metadata.labels}' 2>/dev/null)
      if echo "$LABELS" | grep -q "no-autodelete"; then
        echo "Namespace protected, skipping deletion"
        exit 0
      fi
      helm uninstall app -n $NAMESPACE
      kubectl delete ns $NAMESPACE
```