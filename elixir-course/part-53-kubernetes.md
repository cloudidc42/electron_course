# Part 53: Kubernetes Advanced (Steps 581-600)

## Step 581: Production Deployment Architecture

```yaml
# Full Kubernetes deployment for Elixir/Phoenix app
#
# Architecture:
#   Ingress (nginx/traefik)
#     └── Service (ClusterIP)
#           └── Deployment (Phoenix pods)
#                 ├── initContainer (migration)
#                 └── container (Phoenix app)
#   StatefulSet (PostgreSQL)
#   StatefulSet (Redis)
#   HorizontalPodAutoscaler
#   PodDisruptionBudget

# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    name: my-app
```

---

## Step 582: Deployment with Rolling Update

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0      # zero-downtime
  template:
    metadata:
      labels:
        app: my-app
        version: "1.0.0"
    spec:
      serviceAccountName: my-app
      terminationGracePeriodSeconds: 60

      initContainers:
        - name: migrate
          image: my-app:1.0.0
          command: ["bin/my_app", "eval", "MyApp.Release.migrate()"]
          envFrom:
            - secretRef:
                name: my-app-secrets

      containers:
        - name: my-app
          image: my-app:1.0.0
          ports:
            - containerPort: 4000
          envFrom:
            - configMapRef:
                name: my-app-config
            - secretRef:
                name: my-app-secrets
          resources:
            requests:
              cpu:    "250m"
              memory: "256Mi"
            limits:
              cpu:    "1000m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: 4000
            initialDelaySeconds: 10
            periodSeconds:       5
            failureThreshold:    3
          livenessProbe:
            httpGet:
              path: /health
              port: 4000
            initialDelaySeconds: 30
            periodSeconds:       10
            failureThreshold:    3
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
```

---

## Step 583: Horizontal Pod Autoscaler

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: my-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type:               AverageUtilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type:            AverageValue
          averageValue:    400Mi
    - type: Pods
      pods:
        metric:
          name: phoenix_request_queue_depth
        target:
          type:         AverageValue
          averageValue: "100"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type:          Pods
          value:         2
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type:          Pods
          value:         4
          periodSeconds: 30
```

---

## Step 584: Pod Disruption Budget

```yaml
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: my-app
spec:
  minAvailable: 2       # always keep at least 2 pods running
  selector:
    matchLabels:
      app: my-app

---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: my-app
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 4000
  type: ClusterIP

---
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: my-app
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect:       "true"
    nginx.ingress.kubernetes.io/proxy-body-size:    "10m"
    cert-manager.io/cluster-issuer:                  "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [myapp.com]
      secretName: myapp-tls
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

---

## Step 585: ConfigMap & Secrets

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
  namespace: my-app
data:
  PHX_HOST:             "myapp.com"
  PORT:                 "4000"
  POOL_SIZE:            "10"
  LOG_LEVEL:            "info"
  RELEASE_COOKIE:       "my-cluster-cookie"

---
# secret.yaml (in real life, use SealedSecrets or external-secrets)
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secrets
  namespace: my-app
type: Opaque
stringData:
  DATABASE_URL:  "ecto://user:pass@postgres:5432/myapp_prod"
  REDIS_URL:     "redis://redis:6379"
  SECRET_KEY_BASE: "very-long-secret"
  GUARDIAN_SECRET: "jwt-secret"
```

```bash
# Create secrets from env file (don't commit .env to git!)
kubectl create secret generic my-app-secrets \
  --from-env-file=.env.prod \
  --namespace=my-app \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

## Step 586: StatefulSet for PostgreSQL

```yaml
# postgres.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: my-app
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          env:
            - name:  POSTGRES_DB
              value: myapp_prod
            - name:  POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key:  password
          ports:
            - containerPort: 5432
          volumeMounts:
            - name:      postgres-data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu:    "500m"
              memory: "512Mi"
            limits:
              cpu:    "2000m"
              memory: "2Gi"
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 50Gi

---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: my-app
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
  clusterIP: None  # headless for StatefulSet
```

---

## Step 587: Erlang Clustering on Kubernetes

```elixir
# mix.exs
{:libcluster, "~> 3.3"},
{:dns_cluster, "~> 0.1"}

# config/prod.exs
config :libcluster,
  topologies: [
    k8s: [
      strategy: Cluster.Strategy.Kubernetes.DNS,
      config: [
        service: "my-app-headless",
        application_name: "my_app",
        namespace: "my-app",
        polling_interval: 10_000
      ]
    ]
  ]

# application.ex
children = [
  {Cluster.Supervisor, [Application.get_env(:libcluster, :topologies), [name: MyApp.ClusterSupervisor]]}
  | other_children
]

# Headless service for DNS cluster discovery
# kubernetes/headless-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-headless
  namespace: my-app
spec:
  clusterIP: None
  selector:
    app: my-app
  ports:
    - port: 4369  # EPMD
      name: epmd
    - port: 9000
      name: erlang
```

---

## Step 588: Helm Chart Structure

```
my-app-chart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── configmap.yaml
│   └── _helpers.tpl
└── charts/
    └── postgresql-13.2.0.tgz

# Chart.yaml
apiVersion: v2
name: my-app
version: 1.0.0
appVersion: "1.0.0"
dependencies:
  - name:       postgresql
    version:    "13.x.x"
    repository: "https://charts.bitnami.com/bitnami"
  - name:       redis
    version:    "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
```

```yaml
# values.yaml
replicaCount: 3

image:
  repository: myregistry/my-app
  tag:        "1.0.0"
  pullPolicy: IfNotPresent

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

postgresql:
  enabled:              true
  auth:
    database:           myapp_prod
    existingSecret:     postgres-secret
  primary:
    persistence:
      size: 50Gi

redis:
  enabled:   true
  auth:
    enabled: false
```

```bash
# Install / upgrade
helm upgrade --install my-app ./my-app-chart \
  --namespace my-app \
  --create-namespace \
  --values values.prod.yaml \
  --set image.tag=$IMAGE_TAG
```

---

## Step 589: Network Policies

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-app-network-policy
  namespace: my-app
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - port: 4000
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - port: 6379
    - to: []  # Allow DNS
      ports:
        - port: 53
          protocol: UDP
```

---

## Step 590: CI/CD with GitHub Actions + Kubernetes

```yaml
# .github/workflows/deploy.yml
name: Deploy to Kubernetes

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
    steps:
      - uses: actions/checkout@v4
      - uses: erlef/setup-beam@v1
        with:
          elixir-version: "1.16"
          otp-version:    "26"
      - run: mix deps.get
      - run: mix test
      - run: mix credo --strict
      - run: mix dialyzer

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            myregistry/my-app:${{ github.sha }}
            myregistry/my-app:latest

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBE_CONFIG }}

      - name: Deploy
        run: |
          helm upgrade --install my-app ./k8s/helm \
            --namespace my-app \
            --set image.tag=${{ github.sha }} \
            --wait \
            --timeout 10m

      - name: Verify deployment
        run: kubectl rollout status deployment/my-app -n my-app
```

---

## สรุป Part 53

✅ **Step 581** - Architecture overview  
✅ **Step 582** - Rolling deployment  
✅ **Step 583** - HPA autoscaling  
✅ **Step 584** - Pod disruption budget  
✅ **Step 585** - ConfigMap & Secrets  
✅ **Step 586** - StatefulSet PostgreSQL  
✅ **Step 587** - Erlang clustering  
✅ **Step 588** - Helm charts  
✅ **Step 589** - Network policies  
✅ **Step 590** - CI/CD pipeline  

➡️ [Part 54: Multi-Region & Disaster Recovery](./part-54-multiregion.md)
