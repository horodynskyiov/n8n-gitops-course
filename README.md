n8n GitOps Deployment (Flux CD)

Containerization (Stage 1)

У даному проєкті використано open-source застосунок n8n.

Застосунок розгортається на основі офіційного Docker-образу, що підтримується розробниками n8n:

Repository: n8nio/n8n

Відповідно до вимог курсової роботи, для open-source продуктів дозволено використання готових офіційних образів без написання власного Dockerfile.
Таким чином, етап Containerization виконано згідно з Варіантом Б.


Repository Structure

n8n-gitops-course
│   README.md
│
├── apps
│   ├── production
│   │   ├── cnpg-cluster.yaml
│   │   ├── db-secret-app.yaml
│   │   ├── db-secret-superuser.yaml
│   │   ├── redis.yaml
│   │   ├── n8n-helmrelease.yaml
│   │   └── kustomization.yaml
│   │
│   └── staging
│       ├── cnpg-cluster.yaml
│       ├── db-secret-app.yaml
│       ├── db-secret-superuser.yaml
│       ├── n8n-helmrelease.yaml
│       └── kustomization.yaml
│
├── clusters
│   └── my-cluster
│       ├── infrastructure-kustomization.yaml
│       ├── production-kustomization.yaml
│       ├── staging-kustomization.yaml
│       └── flux-system
│           ├── gotk-components.yaml
│           ├── gotk-sync.yaml
│           └── kustomization.yaml
│
└── infrastructure
    ├── bitnami
    ├── cnpg-operator
    └── n8n-chart


GitOps & Flux CD Status

Flux CD ініціалізовано та синхронізовано з GitHub-репозиторієм.
Усі Kustomization успішно застосовані:

flux get kustomizations -A

| Namespace   | Name           | Ready |
| ----------- | -------------- | ----- |
| flux-system | flux-system    | True  |
| flux-system | infrastructure | True  |
| flux-system | staging        | True  |
| flux-system | production     | True  |

Helm Releases

flux get helmreleases -A

| Namespace   | Name          | Ready |
| ----------- | ------------- | ----- |
| flux-system | cnpg-operator | True  |
| staging     | n8n           | True  |
| production  | n8n           | True  |


Production Workload Status

У namespace production розгорнута повна n8n-архітектура:

n8n main
n8n webhook
n8n workers (queue mode)
PostgreSQL (CloudNativePG: primary + replica)
Redis

kubectl get pods -n production

Усі поди знаходяться у стані Running.


Ingress & Networking
Зовнішній доступ реалізовано через Traefik Ingress Controller.

kubectl -n production get ingress

NAME   CLASS     HOSTS       ADDRESS         PORTS
n8n    traefik   n8n.local   192.168.127.2   80

Ingress коректно маршрутизує трафік між main та webhook компонентами.


Access to Application (Local Environment)

Оскільки кластер розгорнуто у Rancher Desktop (WSL2), прямий доступ до LoadBalancer IP
може бути обмежений особливостями локальної мережі.

Гарантований доступ (port-forward)
kubectl -n production port-forward svc/n8n 8080:5678

Після цього редактор доступний у браузері:
http://localhost:8080


Queue Mode & Autoscaling

n8n працює у queue mode з окремими worker-подами.
Для worker-deployment налаштовано Horizontal Pod Autoscaler.

kubectl -n production get hpa

HPA Parameters
Min replicas: 2
Max replicas: 5
Metrics: CPU + Memory
Observed Behavior
HPA автоматично масштабував worker-поди з 2 → 5
Причина масштабування — перевищення memory utilization (~88–92% від request)
Це підтверджує коректну роботу autoscaling у production-середовищі.


Application Logs
n8n main
kubectl -n production logs deploy/n8n

Застосунок успішно стартує
Task Broker ініціалізовано
JS Task Runner зареєстровано
Editor доступний

n8n webhook
kubectl -n production logs deploy/n8n-webhook

Webhook listener активний та очікує запитів.

