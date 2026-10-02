# Part 68: Advanced Deployment Strategies (Steps 741-760)

## Step 741: Zero-Downtime Deployments

```
Deployment Strategies:

Rolling Update:
  v1 v1 v1 v1   →   v2 v1 v1 v1   →   v2 v2 v1 v1   →   v2 v2 v2 v2
  (replace one at a time)

Blue-Green:
  Load Balancer
      │
  ┌───┴───┐
  │ Blue  │ ← Live traffic
  │  v1   │
  └───────┘
  ┌───────┐
  │ Green │ ← Deploy v2, then switch
  │  v2   │
  └───────┘

Canary:
  100% traffic → v1
  10% traffic  → v2 (canary)
  Monitor metrics, then gradually shift

Phoenix hot code upgrade:
  - BEAM supports code hot-swapping without restarts
  - Requires careful OTP release management
```

---

## Step 742: Mix Releases

```bash
# Create a production release
MIX_ENV=prod mix release

# Release structure
_build/prod/rel/my_app/
  bin/
    my_app        # Control script
    my_app.bat    # Windows
  erts-X.X.X/    # Embedded Erlang runtime
  lib/            # All compiled deps
  releases/
    1.0.0/
      my_app.tar.gz
      sys.config
      vm.args
```

```elixir
# config/runtime.exs - evaluated at runtime
import Config

config :my_app, MyApp.Repo,
  url:      System.fetch_env!("DATABASE_URL"),
  pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10")

config :my_app, MyAppWeb.Endpoint,
  url:         [host: System.fetch_env!("PHX_HOST"), port: 443, scheme: "https"],
  http:        [port: String.to_integer(System.get_env("PORT") || "4000")],
  secret_key_base: System.fetch_env!("SECRET_KEY_BASE")

# rel/env.sh.eex - runtime environment setup
export RELEASE_DISTRIBUTION=name
export RELEASE_NODE="<%= @release.name %>@$(hostname -f)"
```

```elixir
# mix.exs - release config
releases: [
  my_app: [
    version: "1.0.0",
    include_executables_for: [:unix],
    applications: [runtime_tools: :permanent],
    steps: [:assemble, :tar],
    overlays: [{"rel/overlays", "."}]
  ]
]
```

---

## Step 743: Docker Multi-Stage Build

```dockerfile
# Dockerfile
FROM elixir:1.17-alpine AS build

# Install build deps
RUN apk add --no-cache build-base git nodejs npm

WORKDIR /app

# Install Hex + Rebar
RUN mix local.hex --force && mix local.rebar --force

# Fetch dependencies
COPY mix.exs mix.lock ./
RUN MIX_ENV=prod mix deps.get --only prod

# Build assets
COPY assets assets
RUN npm --prefix assets ci --progress=false --no-audit --loglevel=error
RUN npm run --prefix assets deploy

# Compile
COPY config config
COPY lib lib
COPY priv priv
RUN MIX_ENV=prod mix do compile, phx.digest, release

# Runtime stage
FROM alpine:3.19 AS runtime

RUN apk add --no-cache libstdc++ openssl ncurses-libs libgcc

WORKDIR /app

# Copy release from build stage
COPY --from=build /app/_build/prod/rel/my_app ./

ENV PHX_SERVER=true
EXPOSE 4000

CMD ["/app/bin/my_app", "start"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "4000:4000"
    environment:
      DATABASE_URL: "ecto://postgres:password@db/my_app_prod"
      SECRET_KEY_BASE: "${SECRET_KEY_BASE}"
      PHX_HOST: "localhost"
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

---

## Step 744: Kubernetes Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0   # zero downtime
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: ghcr.io/myorg/my-app:latest
          ports:
            - containerPort: 4000
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: my-app-secrets
                  key: database_url
            - name: SECRET_KEY_BASE
              valueFrom:
                secretKeyRef:
                  name: my-app-secrets
                  key: secret_key_base
            - name: PHX_HOST
              valueFrom:
                configMapKeyRef:
                  name: my-app-config
                  key: phx_host
          readinessProbe:
            httpGet:
              path: /health
              port: 4000
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 4000
            initialDelaySeconds: 30
            periodSeconds: 10
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"

---
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 4000

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## Step 745: Health Check Endpoint

```elixir
defmodule MyAppWeb.HealthController do
  use MyAppWeb, :controller

  def check(conn, _params) do
    checks = %{
      database:  check_database(),
      redis:     check_redis(),
      disk:      check_disk()
    }

    status = if Enum.all?(checks, fn {_, v} -> v == :ok end), do: :ok, else: :degraded

    http_status = if status == :ok, do: 200, else: 503

    conn
    |> put_status(http_status)
    |> json(%{
      status:    status,
      timestamp: DateTime.utc_now(),
      checks:    checks,
      version:   Application.spec(:my_app, :vsn)
    })
  end

  defp check_database do
    case MyApp.Repo.query("SELECT 1") do
      {:ok, _} -> :ok
      _        -> :error
    end
  end

  defp check_redis do
    case Redix.command(:redix, ["PING"]) do
      {:ok, "PONG"} -> :ok
      _             -> :error
    end
  end

  defp check_disk do
    case :disksup.get_disk_data() do
      [] -> :error
      _  -> :ok
    end
  end
end
```

---

## Step 746: Database Migrations in Production

```elixir
# lib/my_app/release.ex
defmodule MyApp.Release do
  @app :my_app

  def migrate do
    load_app()
    for repo <- repos() do
      {:ok, _, _} = Ecto.Migrator.with_repo(repo, &Ecto.Migrator.run(&1, :up, all: true))
    end
  end

  def rollback(repo, version) do
    load_app()
    {:ok, _, _} = Ecto.Migrator.with_repo(repo, &Ecto.Migrator.run(&1, :down, to: version))
  end

  defp repos, do: Application.fetch_env!(@app, :ecto_repos)

  defp load_app do
    Application.load(@app)
  end
end

# Run migrations before server starts
# config/runtime.exs
if System.get_env("RUN_MIGRATIONS") == "true" do
  Application.ensure_all_started(:my_app)
  MyApp.Release.migrate()
end
```

```yaml
# k8s/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-app-migrate
  annotations:
    helm.sh/hook: pre-upgrade,pre-install
    helm.sh/hook-weight: "1"
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: ghcr.io/myorg/my-app:latest
          command: ["/app/bin/my_app", "eval", "MyApp.Release.migrate()"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: my-app-secrets
                  key: database_url
      restartPolicy: Never
```

---

## Step 747: Blue-Green Deployment

```elixir
# lib/my_app_web/controllers/deployment_controller.ex
defmodule MyAppWeb.DeploymentController do
  use MyAppWeb, :controller

  def status(conn, _params) do
    slot = System.get_env("DEPLOY_SLOT", "blue")
    version = Application.spec(:my_app, :vsn) |> to_string()
    
    json(conn, %{slot: slot, version: version})
  end
end
```

```bash
#!/bin/bash
# deploy.sh - Blue-Green deployment script

CURRENT=$(kubectl get svc my-app -o jsonpath='{.spec.selector.slot}')
NEW_SLOT=$([ "$CURRENT" = "blue" ] && echo "green" || echo "blue")
NEW_VERSION=$1

echo "Current slot: $CURRENT, deploying to: $NEW_SLOT"

# Deploy to new slot
kubectl set image deployment/my-app-$NEW_SLOT my-app=ghcr.io/myorg/my-app:$NEW_VERSION

# Wait for rollout
kubectl rollout status deployment/my-app-$NEW_SLOT --timeout=5m

# Run smoke tests
if ! curl -f http://my-app-$NEW_SLOT.internal/health; then
  echo "Smoke tests failed!"
  exit 1
fi

# Switch traffic
kubectl patch svc my-app -p "{\"spec\":{\"selector\":{\"slot\":\"$NEW_SLOT\"}}}"

echo "Traffic switched to $NEW_SLOT"
```

---

## Step 748: Feature Flags

```elixir
defmodule MyApp.FeatureFlags do
  # Use FunWithFlags library
  # mix.exs: {:fun_with_flags, "~> 1.12"}
  
  def enabled?(flag_name, actor \\ nil) do
    FunWithFlags.enabled?(flag_name, for: actor)
  end

  def enable(flag_name),              do: FunWithFlags.enable(flag_name)
  def disable(flag_name),             do: FunWithFlags.disable(flag_name)
  def enable_for(flag_name, actor),   do: FunWithFlags.enable(flag_name, for_actor: actor)
  def enable_percentage(flag_name, pct), do: FunWithFlags.enable(flag_name, percentage_of_time: pct)
end

# config/config.exs
config :fun_with_flags, :persistence,
  adapter: FunWithFlags.Store.Persistent.Ecto,
  repo:    MyApp.Repo

config :fun_with_flags, :cache,
  enabled: true,
  ttl:     900  # 15 minutes

# Usage in controller
def create(conn, params) do
  if MyApp.FeatureFlags.enabled?(:new_checkout, for: conn.assigns.current_user) do
    MyApp.NewCheckout.process(params)
  else
    MyApp.LegacyCheckout.process(params)
  end
end
```

---

## Step 749: Canary Deployments with Nginx

```nginx
# nginx.conf
upstream my_app_stable {
  server app-stable:4000 weight=9;  # 90% traffic
}

upstream my_app_canary {
  server app-canary:4000 weight=1;  # 10% traffic
}

# Use consistent hashing for session stickiness
upstream my_app_mixed {
  hash $cookie_session_id consistent;
  server app-stable:4000;
  server app-canary:4000;
}

server {
  listen 80;
  
  location / {
    proxy_pass http://my_app_mixed;
    proxy_set_header X-Real-IP $remote_addr;
  }
  
  # A/B test endpoint
  location /new-feature {
    set $upstream my_app_stable;
    if ($http_x_canary = "true") {
      set $upstream my_app_canary;
    }
    proxy_pass http://$upstream;
  }
}
```

---

## Step 750: Deployment Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

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
          otp-version: '27.0'
          elixir-version: '1.17.0'
      - run: mix deps.get
      - run: mix test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to staging
        run: |
          kubectl set image deployment/my-app \
            my-app=ghcr.io/${{ github.repository }}:${{ github.sha }}

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy to production
        run: |
          kubectl set image deployment/my-app \
            my-app=ghcr.io/${{ github.repository }}:${{ github.sha }}
          kubectl rollout status deployment/my-app
```

---

## สรุป Part 68

✅ **Step 741** - Zero-downtime strategies  
✅ **Step 742** - Mix releases  
✅ **Step 743** - Docker multi-stage build  
✅ **Step 744** - Kubernetes deployment  
✅ **Step 745** - Health check endpoint  
✅ **Step 746** - DB migrations in production  
✅ **Step 747** - Blue-green deployment  
✅ **Step 748** - Feature flags  
✅ **Step 749** - Canary with Nginx  
✅ **Step 750** - Deployment pipeline  

➡️ [Part 69: Event Sourcing & CQRS](./part-69-event-sourcing.md)
