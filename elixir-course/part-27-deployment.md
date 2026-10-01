# Part 27: Deployment และ DevOps (Steps 291-310)

## Step 291: Mix Releases

```elixir
# mix.exs
def project do
  [
    app: :my_app,
    version: "1.0.0",
    releases: [
      my_app: [
        include_executables_for: [:unix],
        steps: [:assemble, :tar]
      ]
    ]
  ]
end

# Build release
# MIX_ENV=prod mix release

# Release structure:
# _build/prod/rel/my_app/
#   bin/my_app         ← start/stop/remote_console
#   lib/               ← compiled BEAM files
#   releases/          ← version metadata

# Commands:
# bin/my_app start       # foreground
# bin/my_app daemon      # background
# bin/my_app remote      # remote IEx session
# bin/my_app eval "MyApp.do_something()"
# bin/my_app stop
```

---

## Step 292: Runtime Configuration

```elixir
# config/runtime.exs - evaluated at runtime (after release build)
import Config

if config_env() == :prod do
  config :my_app, MyApp.Repo,
    url: System.fetch_env!("DATABASE_URL"),
    pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10"),
    ssl: true,
    ssl_opts: [verify: :verify_none]
  
  config :my_app, MyAppWeb.Endpoint,
    url: [host: System.fetch_env!("PHX_HOST"), port: 443, scheme: "https"],
    http: [port: String.to_integer(System.get_env("PORT") || "4000")],
    secret_key_base: System.fetch_env!("SECRET_KEY_BASE"),
    server: true
  
  config :my_app,
    stripe_secret_key: System.fetch_env!("STRIPE_SECRET_KEY"),
    sendgrid_api_key: System.fetch_env!("SENDGRID_API_KEY")
end

# Generate SECRET_KEY_BASE:
# mix phx.gen.secret
```

---

## Step 293: Docker

```dockerfile
# Dockerfile (multi-stage build)
ARG ELIXIR_VERSION=1.16.0
ARG OTP_VERSION=26.2.1
ARG DEBIAN_VERSION=bookworm-20240130-slim

FROM hexpm/elixir:${ELIXIR_VERSION}-erlang-${OTP_VERSION}-debian-${DEBIAN_VERSION} AS build

# Install build deps
RUN apt-get update -y && apt-get install -y build-essential git \
    && apt-get clean && rm -f /var/lib/apt/lists/*_*

WORKDIR /app

# Install hex + rebar
RUN mix local.hex --force && mix local.rebar --force

ENV MIX_ENV="prod"

# Install mix deps
COPY mix.exs mix.lock ./
RUN mix deps.get --only $MIX_ENV

# Compile deps first (cached layer)
RUN mkdir config
COPY config/config.exs config/${MIX_ENV}.exs config/
RUN mix deps.compile

# Build assets
COPY priv priv
COPY assets assets
RUN mix assets.deploy

# Compile and build release
COPY lib lib
RUN mix compile
COPY config/runtime.exs config/
RUN mix release

# Runtime image - small base
FROM debian:${DEBIAN_VERSION}

RUN apt-get update -y && apt-get install -y libstdc++6 openssl libncurses5 locales ca-certificates \
    && apt-get clean && rm -f /var/lib/apt/lists/*_*

RUN sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen && locale-gen
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

WORKDIR "/app"
RUN chown nobody /app

ENV MIX_ENV="prod"

COPY --from=build --chown=nobody:root /app/_build/${MIX_ENV}/rel/my_app ./

USER nobody

CMD ["/app/bin/server"]
```

```bash
# docker-compose.yml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: my_app_prod
    volumes:
      - postgres_data:/var/lib/postgresql/data

  app:
    build: .
    ports:
      - "4000:4000"
    environment:
      DATABASE_URL: "ecto://postgres:postgres@db/my_app_prod"
      SECRET_KEY_BASE: "${SECRET_KEY_BASE}"
      PHX_HOST: "localhost"
      PORT: "4000"
    depends_on:
      - db

volumes:
  postgres_data:
```

---

## Step 294: CI/CD ด้วย GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
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
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    env:
      MIX_ENV: test
      DATABASE_URL: postgresql://postgres:postgres@localhost/my_app_test
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Elixir
        uses: erlef/setup-beam@v1
        with:
          elixir-version: '1.16'
          otp-version: '26'
      
      - name: Cache deps
        uses: actions/cache@v3
        with:
          path: deps
          key: ${{ runner.os }}-mix-${{ hashFiles('**/mix.lock') }}
      
      - name: Cache build
        uses: actions/cache@v3
        with:
          path: _build
          key: ${{ runner.os }}-build-${{ hashFiles('**/mix.lock') }}
      
      - name: Install deps
        run: mix deps.get
      
      - name: Check formatting
        run: mix format --check-formatted
      
      - name: Compile (no warnings)
        run: mix compile --warnings-as-errors
      
      - name: Setup DB
        run: mix ecto.setup
      
      - name: Run tests
        run: mix test --cover
      
      - name: Run Credo
        run: mix credo --strict

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        run: docker build -t my_app:${{ github.sha }} .
      
      - name: Push to registry
        run: |
          docker tag my_app:${{ github.sha }} ghcr.io/${{ github.repository }}:latest
          docker push ghcr.io/${{ github.repository }}:latest
```

---

## Step 295: Fly.io Deployment

```toml
# fly.toml
app = "my-elixir-app"
primary_region = "sin"

[build]

[env]
  PHX_HOST = "my-elixir-app.fly.dev"
  PORT = "8080"
  MIX_ENV = "prod"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = true
  auto_start_machines = true
  min_machines_running = 0

  [http_service.concurrency]
    type = "connections"
    hard_limit = 1000
    soft_limit = 800

[[vm]]
  memory = "512mb"
  cpu_kind = "shared"
  cpus = 1
```

```bash
# Deploy to Fly.io
fly launch
fly secrets set SECRET_KEY_BASE=$(mix phx.gen.secret)
fly postgres create --name my-app-db
fly postgres attach my-app-db
fly deploy

# Scale
fly scale count 3  # 3 instances
fly scale vm performance-1x  # bigger VM
```

---

## Step 296: Health Checks

```elixir
defmodule MyAppWeb.HealthController do
  use MyAppWeb, :controller
  
  def check(conn, _params) do
    checks = %{
      database: check_database(),
      redis:    check_redis(),
      memory:   check_memory()
    }
    
    status = if Enum.all?(Map.values(checks), &(&1.status == :ok)),
      do: :ok,
      else: :degraded
    
    http_status = if status == :ok, do: 200, else: 503
    
    conn
    |> put_status(http_status)
    |> json(%{status: status, checks: checks, timestamp: DateTime.utc_now()})
  end
  
  defp check_database do
    case Ecto.Adapters.SQL.query(MyApp.Repo, "SELECT 1", []) do
      {:ok, _} -> %{status: :ok}
      {:error, e} -> %{status: :error, message: Exception.message(e)}
    end
  rescue
    e -> %{status: :error, message: Exception.message(e)}
  end
  
  defp check_memory do
    memory = :erlang.memory()
    total_mb = div(memory[:total], 1_048_576)
    
    if total_mb < 400 do
      %{status: :ok, total_mb: total_mb}
    else
      %{status: :warning, total_mb: total_mb}
    end
  end
  
  defp check_redis do
    case Redix.command(:redis, ["PING"]) do
      {:ok, "PONG"} -> %{status: :ok}
      _ -> %{status: :error}
    end
  end
end

# router.ex
scope "/health", MyAppWeb do
  pipe_through :api
  get "/", HealthController, :check
  get "/ready", HealthController, :check
  get "/live", HealthController, :live
end
```

---

## Step 297: Migrations ใน Production

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
    Application.ensure_all_started(:ssl)
    Application.ensure_all_started(:postgrex)
  end
end

# Run in Docker/release:
# bin/my_app eval "MyApp.Release.migrate()"
```

```yaml
# GitHub Actions: run migration before deploy
- name: Run migrations
  run: |
    fly ssh console -C "/app/bin/my_app eval MyApp.Release.migrate()"
```

---

## Step 298: Secrets Management

```bash
# .env (never commit!)
DATABASE_URL=ecto://user:pass@localhost/my_app_prod
SECRET_KEY_BASE=<64 char random>
STRIPE_SECRET_KEY=sk_live_...

# Load in dev
source .env

# Fly.io secrets
fly secrets set DATABASE_URL="$DATABASE_URL"
fly secrets set SECRET_KEY_BASE="$(mix phx.gen.secret)"

# Heroku
heroku config:set DATABASE_URL="$DATABASE_URL"

# AWS Secrets Manager
defmodule MyApp.Secrets do
  def fetch(key) do
    ExAws.SecretsManager.get_secret_value(key)
    |> ExAws.request!()
    |> Map.get("SecretString")
    |> Jason.decode!()
  end
end
```

---

## Step 299: Zero-Downtime Deploys

```elixir
# Hot code reloading (within same major version)
defmodule MyApp.Upgrader do
  def upgrade(vsn) do
    :release_handler.unpack_release(vsn)
    :release_handler.install_release(vsn)
    :release_handler.make_permanent(vsn)
  end
end

# Phoenix endpoint: automatic reconnect for WebSockets
# LiveView automatically reconnects on deploy

# Rolling deploys (Fly.io)
# fly.toml: strategy = "rolling"

# Blue-green with Nginx:
# upstream app {
#   server app-blue:4000;   ← active
# }
# Switch:
# upstream app {
#   server app-green:4000;  ← after deploy
# }
```

---

## Step 300: Logging

```elixir
# config/prod.exs
config :logger,
  level: :info,
  backends: [:console]

config :logger, :console,
  format: "$time $metadata[$level] $message\n",
  metadata: [:request_id, :user_id, :trace_id]

# Structured logging with LoggerJSON
config :logger,
  backends: [LoggerJSON]

config :logger_json, :backend,
  formatter: LoggerJSON.Formatters.GoogleCloud,
  metadata: :all

# Usage
require Logger

Logger.info("Order created", order_id: order.id, user_id: user.id, total: order.total)
Logger.warning("Slow query detected", duration_ms: 500, query: sql)
Logger.error("Payment failed", error: reason, user_id: user.id)

# Request ID plug (router.ex)
plug Plug.RequestId
plug Plug.Logger, log: :info
```

---

## สรุป Part 27

✅ **Step 291** - Mix releases  
✅ **Step 292** - Runtime configuration  
✅ **Step 293** - Docker multi-stage  
✅ **Step 294** - GitHub Actions CI/CD  
✅ **Step 295** - Fly.io deployment  
✅ **Step 296** - Health checks  
✅ **Step 297** - Production migrations  
✅ **Step 298** - Secrets management  
✅ **Step 299** - Zero-downtime deploys  
✅ **Step 300** - Structured logging  

➡️ [Part 28: GraphQL ด้วย Absinthe](./part-28-graphql.md)
