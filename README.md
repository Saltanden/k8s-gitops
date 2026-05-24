# k8s-gitops

## Опис проєкту

GitOps-інфраструктура для розгортання **course-app** (Python FastAPI) з **Dragonfly** як базою даних (Redis-compatible), керованою через **Dragonfly Operator**.

Будь-яка зміна в цьому репозиторії автоматично відображається в кластері без ручного `kubectl apply`.

## Стек

| Компонент | Технологія |
|-----------|-----------|
| Застосунок | course-app (Python FastAPI) |
| База даних | Dragonfly (Redis-compatible) |
| Оператор БД | Dragonfly Operator |
| Образ | ghcr.io/saltanden/course-app:v1 |
| Templating | Helm (власний чарт) |
| GitOps | Flux CD v2 |
| Оркестрація | Kubernetes (minikube) |

## Структура репозиторію

```
k8s-gitops/
├── charts/
│   └── course-app/            # Власний Helm чарт
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── ingress.yaml
│           └── hpa.yaml
├── infrastructure/
│   └── controllers/
│       └── dragonfly/         # Dragonfly Operator (встановлений через kubectl)
├── apps/
│   └── overlays/
│       ├── staging/           # 1 репліка, без limits
│       └── production/        # HPA 2-5, resources limits, Dragonfly x2
└── clusters/
    └── my-cluster/            # Flux Kustomization маніфести
```

## Середовища

| | Staging | Production |
|--|---------|-----------|
| Namespace | staging | production |
| Репліки course-app | 1 (фіксовано) | 2-5 (HPA) |
| Dragonfly | 1 репліка | 2 репліки |
| Ingress | app.staging.local | app.local |
| Resources | відсутні | requests + limits |
| HPA | ні | так (CPU 70%) |

## Результати

### flux get kustomizations

```
NAME            REVISION              SUSPENDED  READY  MESSAGE
app-production  main@sha1:bf2406ab    False      True   Applied revision: main@sha1:bf2406ab
app-staging     main@sha1:bf2406ab    False      True   Applied revision: main@sha1:bf2406ab
flux-system     main@sha1:bf2406ab    False      True   Applied revision: main@sha1:bf2406ab
infrastructure  main@sha1:bf2406ab    False      True   Applied revision: main@sha1:bf2406ab
```

### flux get helmreleases -A

```
NAMESPACE   NAME        REVISION  READY  MESSAGE
production  course-app  1.0.1     True   Helm install succeeded
staging     course-app  1.0.1     True   Helm install succeeded
```

### kubectl get pods -A (staging та production)

```
staging:
  course-app-xxx   1/1  Running  (1 pod)
  dragonfly-0      1/1  Running

production:
  course-app-xxx   1/1  Running  (3 pods)
  course-app-xxx   1/1  Running
  course-app-xxx   1/1  Running
  dragonfly-0      1/1  Running
  dragonfly-1      1/1  Running
```

### kubectl get ingress -A

```
NAMESPACE   NAME                CLASS  HOSTS              ADDRESS
production  course-app-ingress  nginx  app.local          192.168.49.2
staging     course-app-ingress  nginx  app.staging.local  192.168.49.2
```

### kubectl get hpa -n production

```
NAME            REFERENCE             TARGETS            MINPODS  MAXPODS  REPLICAS
course-app-hpa  Deployment/course-app cpu: <unknown>/70% 2        5        3
```

## Self-Healing

При видаленні deployment (`kubectl delete deployment course-app -n production`) Flux автоматично відновлює його протягом хвилини через reconciliation loop.

## GitOps цикл

1. Зміна в Git (push)
2. Flux Source Controller клонує репозиторій
3. Kustomize Controller застосовує маніфести
4. Helm Controller встановлює/оновлює HelmRelease
5. Kubernetes застосовує бажаний стан