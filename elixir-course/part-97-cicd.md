# Part 97: CI/CD & Deployment (Steps 1031-1040)

## Step 1031: GitHub Actions CI

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  MIX_ENV: test
  ELIXIR_VERSION: "1.17"
  OTP_VERSION: "27"

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: myapp_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports: ["5432:5432"]

      redis:
        image: redis:7
        ports: ["6379:6379"]

    steps:
      - uses: actions/checkout@v4
      
      - uses: erlef/setup-beam@v1
        with:
          elixir-version: ${{ env.ELIXIR_VERSION }}
          otp-version: ${{ env.OTP_VERSION }}

      - name: Restore deps cache
        uses: actions/cache@v4
        with:
          path: deps
          key: ${{ runner.os }}-mix-${{ hashFiles('**/mix.lock') }}

      - name: Restore build cache
        uses: actions/cache@v4
        with:
          path: _build
          key: ${{ runner.os }}-build-${{ hashFiles('**/mix.lock') }}

      - run: mix deps.get
      - run: mix compile --warnings-as-errors
      - run: mix format --check-formatted
      - run: mix credo --strict
      - run: mix test --cover
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: ./cover/excoveralls.json
```

---

## Step 1032: Docker Setup

```dockerfile
# Dockerfile
FROM hexpm/elixir:1.17.3-erlang-27.0-debian-bookworm-20240904-slim AS builder

WORKDIR /app
ENV MIX_ENV=prod

RUN apt-get update && apt-get install -y build-essential git && \
    mix local.hex --force && \
    mix local.rebar --force

COPY mix.exs mix.lock ./
RUN mix deps.get --only prod

COPY config/config.exs config/prod.exs config/
RUN mix deps.compile

COPY assets/ assets/
RUN mix assets.deploy

COPY . .
RUN mix compile && mix release

# --- Runtime image ---
FROM debian:bookworm-slim AS runtime

RUN apt-get update && apt-get install -y libssl3 libncurses6 curl && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY --from=builder /app/_build/prod/rel/my_app ./

RUN useradd -r -s /bin/false appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 4000
HEALTHCHECK --interval=30s --timeout=10s CMD curl -f http://localhost:4000/health || exit 1

CMD ["/app/bin/my_app", "start"]
```

```yaml
# docker-compose.yml
version: "3.8"
services:
  app:
    build: .
    ports: ["4000:4000"]
    environment:
      DATABASE_URL:    postgres://postgres:postgres@db:5432/myapp
      REDIS_URL:       redis://redis:6379
      SECRET_KEY_BASE: "${SECRET_KEY_BASE}"
      PHX_HOST:        localhost
    depends_on:
      db:    {condition: service_healthy}
      redis: {condition: service_started}

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB:       myapp
    volumes: ["postgres_data:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout:  5s
      retries:  5

  redis:
    image: redis:7-alpine

volumes:
  postgres_data:
```

---

## Step 1033: Kubernetes Deployment

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
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: myregistry.io/my-app:1.0.0
        ports:
        - containerPort: 4000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: my-app-secrets
              key: database_url
        - name: RELEASE_DISTRIBUTION
          value: name
        - name: RELEASE_NODE
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        resources:
          requests: {cpu: 100m, memory: 256Mi}
          limits:   {cpu: 500m, memory: 512Mi}
        livenessProbe:
          httpGet: {path: /health/live, port: 4000}
          initialDelaySeconds: 15
          periodSeconds: 20
        readinessProbe:
          httpGet: {path: /health/ready, port: 4000}
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 4000
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts: [myapp.com]
    secretName: my-app-tls
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app
            port: {number: 80}
```

---

## Step 1034: Database Migrations in CI/CD

```elixir
# rel/overlays/bin/migrate
#!/bin/sh
exec /app/bin/my_app eval MyApp.Release.migrate

# lib/my_app/release.ex
defmodule MyApp.Release do
  @app :my_app

  def migrate do
    load_app()
    {:ok, _, _} = Ecto.Migrator.with_repo(MyApp.Repo, fn repo ->
      Ecto.Migrator.run(repo, :up, all: true)
    end)
  end

  def rollback(version) do
    load_app()
    {:ok, _, _} = Ecto.Migrator.with_repo(MyApp.Repo, fn repo ->
      Ecto.Migrator.run(repo, :down, to: version)
    end)
  end

  defp load_app do
    Application.load(@app)
    Application.ensure_all_started(:ssl)
    Application.ensure_all_started(:postgrex)
    Application.ensure_all_started(:ecto_sql)
  end
end
```

```yaml
# k8s/migrate-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-app-migrate-${VERSION}
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: myregistry.io/my-app:${VERSION}
        command: ["/app/bin/migrate"]
        envFrom:
        - secretRef:
            name: my-app-secrets
      restartPolicy: Never
  backoffLimit: 3
```

---

## Step 1035: Blue/Green Deployments

```elixir
defmodule MyApp.Deployment.BlueGreen do
  # Blue/Green deployment via load balancer switching

  def switch_traffic(to_color) when to_color in [:blue, :green] do
    update_load_balancer_target(to_color)
    
    # Verify new deployment is healthy
    case verify_health(to_color, timeout: 60_000) do
      :ok ->
        Logger.info("Successfully switched to #{to_color}")
        :ok
      {:error, reason} ->
        Logger.error("Switch failed, rolling back: #{inspect(reason)}")
        rollback(to_color)
        {:error, :rollback_performed}
    end
  end

  defp update_load_balancer_target(color) do
    # Update Kubernetes service selector
    service_name = "my-app-#{color}"
    
    System.cmd("kubectl", [
      "patch", "service", "my-app",
      "--type=json",
      "--patch", Jason.encode!([%{
        op:    "replace",
        path:  "/spec/selector/color",
        value: to_string(color)
      }])
    ])
  end

  defp verify_health(color, opts) do
    deadline = System.monotonic_time(:millisecond) + opts[:timeout]
    service  = "my-app-#{color}"
    
    Stream.repeatedly(fn ->
      Process.sleep(2000)
      check_health(service)
    end)
    |> Stream.take_while(fn
      :ok -> false
      _ when System.monotonic_time(:millisecond) < deadline -> true
      _ -> false
    end)
    |> Stream.run()
    
    check_health(service)
  end

  defp check_health(service) do
    case Req.get("http://#{service}/health") do
      {:ok, %{status: 200}} -> :ok
      _                     -> {:error, :unhealthy}
    end
  end

  defp rollback(:blue), do: update_load_balancer_target(:green)
  defp rollback(:green), do: update_load_balancer_target(:blue)
end
```

---

## Step 1036: Environment Configuration

```elixir
# config/runtime.exs - Full production configuration
import Config

if config_env() == :prod do
  # Required configuration
  database_url = System.fetch_env!("DATABASE_URL")
  secret_key   = System.fetch_env!("SECRET_KEY_BASE")
  host         = System.fetch_env!("PHX_HOST")

  config :my_app, MyApp.Repo,
    url:             database_url,
    pool_size:       System.get_env("DB_POOL_SIZE", "10") |> String.to_integer(),
    ssl:             true,
    ssl_opts: [
      verify:    :verify_peer,
      cacertfile: System.get_env("DB_CA_CERT_PATH", "/etc/ssl/certs/ca-certificates.crt")
    ]

  config :my_app, MyAppWeb.Endpoint,
    http:           [port: 4000],
    url:            [host: host, port: 443, scheme: "https"],
    secret_key_base: secret_key,
    cache_static_manifest: "priv/static/cache_manifest.json"

  config :my_app, :redis,
    url: System.fetch_env!("REDIS_URL")

  config :my_app, :clustering,
    strategy:    Cluster.Strategy.Kubernetes,
    kubernetes_ip_lookup_mode: :pods,
    kubernetes_namespace: System.get_env("RELEASE_NAMESPACE", "production"),
    kubernetes_selector: "app=my-app"
end
```

---

## Step 1037: Logging Setup

```elixir
# config/prod.exs
config :logger,
  level:    :info,
  backends: [:console]

config :logger, :console,
  format: "$time $metadata[$level] $message\n",
  metadata: [:request_id, :user_id, :trace_id]

# Structured JSON logging
defmodule MyApp.Logger.JSONFormatter do
  def format(level, message, timestamp, metadata) do
    log = %{
      timestamp: format_timestamp(timestamp),
      level:     level,
      message:   IO.iodata_to_binary(message),
      request_id: Keyword.get(metadata, :request_id),
      user_id:    Keyword.get(metadata, :user_id),
      trace_id:   Keyword.get(metadata, :trace_id),
      service:    "my_app",
      version:    Application.spec(:my_app, :vsn) |> to_string()
    }
    |> Map.reject(fn {_, v} -> is_nil(v) end)

    Jason.encode!(log) <> "\n"
  rescue
    _ -> "#{level} #{message}\n"
  end

  defp format_timestamp({date, {h, m, s, ms}}) do
    {y, mo, d} = date
    "#{y}-#{pad(mo)}-#{pad(d)}T#{pad(h)}:#{pad(m)}:#{pad(s)}.#{ms}Z"
  end

  defp pad(n) when n < 10, do: "0#{n}"
  defp pad(n), do: "#{n}"
end
```

---

## Step 1038: Monitoring & Alerting

```elixir
# mix.exs: {:prometheus_ex, "~> 3.0"}, {:prometheus_plugs, "~> 1.1"}

defmodule MyApp.Metrics do
  use Prometheus.Metric

  def setup do
    Histogram.declare(
      name: :http_request_duration_ms,
      help: "HTTP request duration in milliseconds",
      labels: [:method, :route, :status],
      buckets: [10, 25, 50, 100, 250, 500, 1000, 2500, 5000]
    )

    Counter.declare(
      name: :http_requests_total,
      help: "Total HTTP requests",
      labels: [:method, :route, :status]
    )

    Gauge.declare(
      name: :db_pool_size,
      help: "Database connection pool size"
    )

    Gauge.declare(
      name: :active_users_total,
      help: "Currently active users"
    )
  end

  def record_request(method, route, status, duration_ms) do
    labels = [method, route, to_string(status)]
    Histogram.observe([name: :http_request_duration_ms, labels: labels], duration_ms)
    Counter.inc([name: :http_requests_total, labels: labels])
  end
end

# Metrics plug
defmodule MyAppWeb.Plugs.Metrics do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    start = System.monotonic_time(:millisecond)
    
    register_before_send(conn, fn conn ->
      duration = System.monotonic_time(:millisecond) - start
      route    = Phoenix.Router.route_info(MyAppWeb.Router, conn.method, conn.request_path, "")
      
      MyApp.Metrics.record_request(
        conn.method,
        route[:route] || conn.request_path,
        conn.status,
        duration
      )
      conn
    end)
  end
end
```

---

## Step 1039: Release Management

```elixir
# mix.exs
def project do
  [
    releases: [
      my_app: [
        steps: [:assemble, &copy_extras/1],
        include_executables_for: [:unix],
        include_erts: true,
        strip_beams: Mix.env() == :prod,
        overlays: "rel/overlays",
        config_providers: [
          {Config.Reader, {:system, "RELEASE_ROOT", "/config/runtime.exs"}}
        ]
      ]
    ]
  ]
end

defp copy_extras(release) do
  # Add migration script
  migrate_script = Path.join(release.path, "bin/migrate")
  File.write!(migrate_script, ~s(#!/bin/sh\nexec /app/bin/my_app eval 'MyApp.Release.migrate()'))
  File.chmod!(migrate_script, 0o755)
  release
end

# CHANGELOG tracking
defmodule Mix.Tasks.Release.Changelog do
  use Mix.Task

  def run([version]) do
    # Auto-generate changelog from git commits
    {commits, 0} = System.cmd("git", [
      "log",
      "--pretty=format:%s",
      "v#{previous_version()}..HEAD"
    ])

    changelog_entry = """
    ## #{version} - #{Date.utc_today()}

    #{format_commits(commits)}
    """

    existing = File.read!("CHANGELOG.md")
    File.write!("CHANGELOG.md", changelog_entry <> existing)
    IO.puts "Updated CHANGELOG.md for #{version}"
  end

  defp format_commits(commits) do
    commits
    |> String.split("\n")
    |> Enum.reject(&(&1 == ""))
    |> Enum.map(&"- #{&1}")
    |> Enum.join("\n")
  end

  defp previous_version do
    {tags, _} = System.cmd("git", ["describe", "--tags", "--abbrev=0"])
    String.trim(tags) |> String.trim_leading("v")
  end
end
```

---

## Step 1040: Post-Deployment Verification

```elixir
defmodule MyApp.DeploymentVerification do
  # Smoke tests after deployment

  @tests [
    {:health_check,   "/health/live"},
    {:readiness,      "/health/ready"},
    {:api_auth,       "/api/v1/auth/status"},
    {:product_list,   "/api/v1/products"}
  ]

  def run(base_url) do
    results = Enum.map(@tests, fn {name, path} ->
      result = run_test(base_url, path)
      {name, result}
    end)

    failed = Enum.filter(results, fn {_, r} -> r != :ok end)

    if length(failed) > 0 do
      Logger.error("Post-deployment checks failed: #{inspect(failed)}")
      {:error, failed}
    else
      Logger.info("All post-deployment checks passed!")
      :ok
    end
  end

  defp run_test(base_url, path) do
    case Req.get("#{base_url}#{path}", receive_timeout: 5_000) do
      {:ok, %{status: s}} when s in 200..299 -> :ok
      {:ok, %{status: status}} -> {:error, "HTTP #{status}"}
      {:error, reason} -> {:error, reason}
    end
  end

  # Run as Oban job after deployment
  defmodule VerifyDeploymentJob do
    use Oban.Worker, queue: :maintenance

    def perform(%Oban.Job{args: %{"version" => version}}) do
      base_url = MyAppWeb.Endpoint.url()
      
      case MyApp.DeploymentVerification.run(base_url) do
        :ok ->
          Logger.info("Deployment #{version} verified")
          :ok
        {:error, failures} ->
          Logger.error("Deployment #{version} verification failed: #{inspect(failures)}")
          {:error, "verification_failed"}
      end
    end
  end
end
```

---

## สรุป Part 97

✅ **Step 1031** - GitHub Actions CI  
✅ **Step 1032** - Docker setup  
✅ **Step 1033** - Kubernetes deployment  
✅ **Step 1034** - Database migrations in CI/CD  
✅ **Step 1035** - Blue/green deployments  
✅ **Step 1036** - Environment configuration  
✅ **Step 1037** - Logging setup  
✅ **Step 1038** - Monitoring & alerting  
✅ **Step 1039** - Release management  
✅ **Step 1040** - Post-deployment verification  

➡️ [Part 98: Advanced Patterns](./part-98-patterns.md)
