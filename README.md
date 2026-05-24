# k8s-gitops

## Опис проєкту

GitOps-інфраструктура для розгортання **course-app** (Python FastAPI) з **Dragonfly** як базою даних (Redis-compatible), керованою через **Dragonfly Operator**.

## Стек

| Компонент | Технологія |
|-----------|-----------|
| Застосунок | course-app (Python FastAPI) |
| База даних | Dragonfly (Redis-compatible) |
| Оператор БД | Dragonfly Operator |
| Контейнеризація | Docker, GHCR |
| Templating | Helm (власний чарт) |
| GitOps | Flux CD |
| Оркестрація | Kubernetes (minikube) |

## Структура репозиторію

```
k8s-gitops/
├── charts/course-app/     # Власний Helm чарт
├── infrastructure/
│   └── controllers/
│       └── dragonfly/     # HelmRelease для Dragonfly Operator
├── apps/
│   └── overlays/
│       ├── staging/       # 1 репліка, мінімальні ресурси
│       └── production/    # HPA 2-5 реплік, resources limits
└── clusters/
    └── my-cluster/        # Flux Kustomization маніфести
```

## Середовища

| | Staging | Production |
|--|---------|-----------|
| Namespace | staging | production |
| Репліки | 1 (фіксовано) | 2-5 (HPA) |
| Dragonfly | 1 репліка | 2 репліки |
| Ingress | app.staging.local | app.local |
| Resources | відсутні | requests + limits |

## Перевірка

```bash
# Статус Flux
flux get kustomizations -A
flux get helmreleases -A

# Поди в обох середовищах
kubectl get pods -n staging
kubectl get pods -n production

# Ingress
kubectl get ingress -A

# HPA в production
kubectl get hpa -n production
```
