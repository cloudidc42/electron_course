# Part 31: Docker และ Kubernetes (Steps 331-350)

## Step 331: Docker Compose สำหรับ Development

```yaml
# docker-compose.yml
version: "3.9"

services:
  app:
    build:
      context: .
      target: dev
    ports:
      - "4000:4000"
    volumes:
      - .:/app
      - mix_deps:/app/deps
      - build:/app/_build
    environment:
      MIX_ENV: dev
      DATABASE_URL: ecto://postgres:postgres@db/my_app_dev
      REDIS_URL: redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    command: mix phx.server

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: my_app_dev
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      - db

volumes:
  postgres_data:
  redis_data:
  mix_deps:
  build:
```

---

## Step 332: Multi-stage Dockerfile

```dockerfile
# Dockerfile
ARG ELIXIR_VERSION=1.16.2
ARG OTP_VERSION=26.2.5
ARG DEBIAN_VERSION=bookworm-20240423-slim

# ─── Development ─────────────────────────
FROM hexpm/elixir:${ELIXIR_VERSION}-erlang-${OTP_VERSION}-debian-${DEBIAN_VERSION} AS dev

RUN apt-get update -y && apt-get install -y \
  build-essential git inotify-tools curl \
  && apt-get clean

WORKDIR /app

RUN mix local.hex --force && mix local.rebar --force

ENV MIX_ENV=dev

COPY mix.exs mix.lock ./
RUN mix deps.get

CMD ["mix", "phx.server"]

# ─── Build ───────────────────────────────
FROM dev AS build

ENV MIX_ENV=prod

COPY mix.exs mix.lock ./
RUN mix deps.get --only $MIX_ENV

RUN mkdir config
COPY config/config.exs config/prod.exs config/runtime.exs config/

COPY priv priv
COPY assets assets
RUN mix assets.deploy

COPY lib lib
RUN mix compile

RUN mix release

# ─── Production ──────────────────────────
FROM debian:${DEBIAN_VERSION} AS prod

RUN apt-get update -y && apt-get install -y \
  libstdc++6 openssl libncurses5 locales ca-certificates \
  && apt-get clean

RUN sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen && locale-gen
ENV LANG=en_US.UTF-8 LANGUAGE=en_US:en LC_ALL=en_US.UTF-8

WORKDIR /app
RUN chown nobody /app

COPY --from=build --chown=nobody:root /app/_build/prod/rel/my_app ./

USER nobody

EXPOSE 4000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:4000/health/live || exit 1

CMD ["/app/bin/server"]
```

---

## Step 333: Kubernetes Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-elixir-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-elixir-app
  template:
    metadata:
      labels:
        app: my-elixir-app
    spec:
      containers:
        - name: app
          image: ghcr.io/myorg/my-elixir-app:v1.2.3
          ports:
            - containerPort: 4000
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url
            - name: SECRET_KEY_BASE
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: secret-key-base
            - name: PHX_HOST
              value: "myapp.com"
            - name: PORT
              value: "4000"
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 4000
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 4000
            initialDelaySeconds: 30
            periodSeconds: 15

---
apiVersion: v1
kind: Service
metadata:
  name: my-elixir-app
  namespace: production
spec:
  selector:
    app: my-elixir-app
  ports:
    - port: 80
      targetPort: 4000
  type: ClusterIP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-elixir-app
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  tls:
    - hosts:
        - myapp.com
      secretName: myapp-tls
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-elixir-app
                port:
                  number: 80
```

---

## Step 334: Kubernetes Secrets

```yaml
# k8s/secrets.yaml (encrypted with SOPS)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
stringData:
  database-url: "ecto://user:pass@postgres-service/my_app_prod"
  secret-key-base: "64_char_random_string_here"
  stripe-secret-key: "sk_live_..."

---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  PHX_HOST: "myapp.com"
  POOL_SIZE: "10"
  LOG_LEVEL: "info"
```

```bash
# kubectl commands
kubectl apply -f k8s/
kubectl rollout status deployment/my-elixir-app

# Scale
kubectl scale deployment my-elixir-app --replicas=5

# Rolling update
kubectl set image deployment/my-elixir-app app=ghcr.io/myorg/app:v1.2.4
kubectl rollout undo deployment/my-elixir-app  # rollback

# Run migrations
kubectl run migrate --image=ghcr.io/myorg/app:v1.2.3 \
  --restart=Never \
  --env="DATABASE_URL=$(kubectl get secret app-secrets -o jsonpath='{.data.database-url}' | base64 -d)" \
  -- /app/bin/my_app eval "MyApp.Release.migrate()"
```

---

## Step 335: Distributed Elixir ใน Kubernetes

```elixir
# Elixir nodes can auto-discover each other in K8s using libcluster
# mix.exs
{:libcluster, "~> 3.3"}

# config/prod.exs
config :libcluster,
  topologies: [
    k8s: [
      strategy: Cluster.Strategy.Kubernetes,
      config: [
        kubernetes_node_basename: "my_app",
        kubernetes_selector: "app=my-elixir-app",
        kubernetes_namespace: "production",
        polling_interval: 10_000
      ]
    ]
  ]

# application.ex
children = [
  {Cluster.Supervisor, [Application.get_env(:libcluster, :topologies), [name: MyApp.ClusterSupervisor]]}
]

# Verify cluster in IEx:
# Node.list()
# => [:"my_app@10.0.0.2", :"my_app@10.0.0.3"]
```

---

## Step 336: Horizontal Pod Autoscaling

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-elixir-app
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-elixir-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
```

---

## Step 337: Persistent Storage

```yaml
# k8s/postgres.yaml (StatefulSet)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
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
          image: postgres:16
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
```

---

## Step 338: CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    tags: ["v*.*.*"]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.ref_name }}
            ghcr.io/${{ github.repository }}:latest
  
  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
      
      - name: Configure kubeconfig
        run: echo "${{ secrets.KUBECONFIG }}" | base64 -d > kubeconfig.yaml
      
      - name: Run migrations
        run: |
          kubectl --kubeconfig=kubeconfig.yaml run migrate-${{ github.sha }} \
            --image=ghcr.io/${{ github.repository }}:${{ github.ref_name }} \
            --restart=Never \
            --command -- /app/bin/my_app eval "MyApp.Release.migrate()"
          
          kubectl --kubeconfig=kubeconfig.yaml wait pod/migrate-${{ github.sha }} \
            --for=condition=complete --timeout=120s
      
      - name: Deploy
        run: |
          kubectl --kubeconfig=kubeconfig.yaml set image \
            deployment/my-elixir-app \
            app=ghcr.io/${{ github.repository }}:${{ github.ref_name }}
          
          kubectl --kubeconfig=kubeconfig.yaml rollout status \
            deployment/my-elixir-app --timeout=300s
```

---

## Step 339: Resource Limits

```elixir
# Elixir memory limits
defmodule MyApp.MemoryGuard do
  use GenServer
  
  @check_interval 30_000
  @max_memory_mb 400
  
  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)
  
  def init(_) do
    schedule_check()
    {:ok, %{}}
  end
  
  def handle_info(:check, state) do
    memory_mb = :erlang.memory(:total) |> div(1_048_576)
    
    if memory_mb > @max_memory_mb do
      require Logger
      Logger.warning("High memory usage: #{memory_mb}MB. Requesting GC.")
      :erlang.garbage_collect()
    end
    
    schedule_check()
    {:noreply, state}
  end
  
  defp schedule_check do
    Process.send_after(self(), :check, @check_interval)
  end
end
```

---

## Step 340: Monitoring ด้วย Prometheus

```yaml
# k8s/servicemonitor.yaml (Prometheus Operator)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-elixir-app
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: my-elixir-app
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

```elixir
# Expose metrics endpoint
defmodule MyAppWeb.Router do
  scope "/metrics" do
    get "/", MyAppWeb.MetricsController, :index
  end
end

defmodule MyAppWeb.MetricsController do
  use MyAppWeb, :controller
  
  def index(conn, _params) do
    metrics = TelemetryMetricsPrometheus.Core.scrape()
    
    conn
    |> put_resp_content_type("text/plain; version=0.0.4")
    |> send_resp(200, metrics)
  end
end
```

---

## สรุป Part 31

✅ **Step 331** - Docker Compose for development  
✅ **Step 332** - Multi-stage Dockerfile  
✅ **Step 333** - Kubernetes Deployment  
✅ **Step 334** - K8s Secrets + ConfigMap  
✅ **Step 335** - Distributed Elixir in K8s  
✅ **Step 336** - Horizontal Pod Autoscaling  
✅ **Step 337** - Persistent storage (StatefulSet)  
✅ **Step 338** - CI/CD pipeline  
✅ **Step 339** - Resource limits  
✅ **Step 340** - Prometheus monitoring  

➡️ [Part 32: Distributed Systems](./part-32-distributed.md)
