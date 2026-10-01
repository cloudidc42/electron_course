# Part 39: Observability และ Monitoring (Steps 431-450)

## Step 431: Observability Three Pillars

```
Three Pillars of Observability:
1. Metrics    - quantitative measurements (Prometheus/Grafana)
2. Logs       - event records (Loki/ELK)
3. Traces     - distributed request flow (Jaeger/Zipkin)

Elixir tools:
- Telemetry   - metrics/event instrumentation
- OpenTelemetry - distributed tracing
- Logger      - structured logging
- :observer   - built-in BEAM inspector
```

---

## Step 432: Telemetry

```elixir
# mix.exs
{:telemetry, "~> 1.2"},
{:telemetry_metrics, "~> 0.6"},
{:telemetry_poller, "~> 1.0"}

defmodule MyApp.Telemetry do
  use Supervisor
  import Telemetry.Metrics
  
  def start_link(arg) do
    Supervisor.start_link(__MODULE__, arg, name: __MODULE__)
  end
  
  def init(_arg) do
    children = [
      # VM metrics every 10s
      {:telemetry_poller,
        measurements: [
          {:process_info, event: [:my_app, :memory], name: :total, keys: [:memory]},
          {MyApp.Telemetry, :dispatch_queue_length, []}
        ],
        period: 10_000
      },
      # Prometheus exporter
      {TelemetryMetricsPrometheus.Core, metrics: metrics()}
    ]
    
    Supervisor.init(children, strategy: :one_for_one)
  end
  
  def metrics do
    [
      # HTTP metrics
      counter("phoenix.router_dispatch.stop.count",
        event_name: [:phoenix, :router_dispatch, :stop],
        tags: [:route, :method, :status]
      ),
      summary("phoenix.router_dispatch.stop.duration",
        event_name: [:phoenix, :router_dispatch, :stop],
        unit: {:native, :millisecond},
        tags: [:route]
      ),
      
      # Database metrics
      counter("my_app.repo.query.count",
        event_name: [:my_app, :repo, :query, :stop],
        tags: [:source]
      ),
      summary("my_app.repo.query.duration",
        event_name: [:my_app, :repo, :query, :stop],
        unit: {:native, :millisecond}
      ),
      
      # Custom business metrics
      counter("my_app.orders.placed.count",
        event_name: [:my_app, :orders, :placed]
      ),
      sum("my_app.orders.revenue",
        event_name: [:my_app, :orders, :placed],
        measurement: :amount
      ),
      
      # VM metrics
      last_value("vm.memory.total", unit: {:byte, :megabyte}),
      last_value("vm.total_run_queue_lengths.total"),
      last_value("vm.total_run_queue_lengths.io")
    ]
  end
  
  def dispatch_queue_length do
    :telemetry.execute([:vm, :queue], %{length: :erlang.statistics(:run_queue)}, %{})
  end
end
```

---

## Step 433: Custom Metrics

```elixir
defmodule MyApp.Metrics do
  def track_order_placed(amount, user_id) do
    :telemetry.execute(
      [:my_app, :orders, :placed],
      %{count: 1, amount: amount},
      %{user_id: user_id}
    )
  end
  
  def track_cache(operation, result) do
    :telemetry.execute(
      [:my_app, :cache, operation],
      %{count: 1},
      %{result: to_string(result)}
    )
  end
  
  def measure(event_prefix, fun) when is_function(fun, 0) do
    start = System.monotonic_time()
    
    try do
      result = fun.()
      duration = System.monotonic_time() - start
      
      :telemetry.execute(
        event_prefix ++ [:stop],
        %{duration: duration},
        %{status: :ok}
      )
      
      result
    rescue
      error ->
        duration = System.monotonic_time() - start
        :telemetry.execute(
          event_prefix ++ [:exception],
          %{duration: duration},
          %{kind: :error, reason: error}
        )
        reraise error, __STACKTRACE__
    end
  end
end

# Usage
MyApp.Metrics.measure([:my_app, :payment, :process], fn ->
  process_payment(order)
end)
```

---

## Step 434: Prometheus + Grafana

```yaml
# docker-compose.yml
prometheus:
  image: prom/prometheus:latest
  ports:
    - "9090:9090"
  volumes:
    - ./prometheus.yml:/etc/prometheus/prometheus.yml
  command: --config.file=/etc/prometheus/prometheus.yml

grafana:
  image: grafana/grafana:latest
  ports:
    - "3000:3000"
  environment:
    - GF_SECURITY_ADMIN_PASSWORD=admin
  depends_on:
    - prometheus
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'elixir_app'
    static_configs:
      - targets: ['app:4000']
    metrics_path: '/metrics'

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

```elixir
# Phoenix endpoint for metrics
defmodule MyAppWeb.MetricsPlug do
  import Plug.Conn
  
  def init(opts), do: opts
  
  def call(conn, _opts) do
    if conn.request_path == "/metrics" do
      metrics = TelemetryMetricsPrometheus.Core.scrape()
      
      conn
      |> put_resp_content_type("text/plain; version=0.0.4")
      |> send_resp(200, metrics)
      |> halt()
    else
      conn
    end
  end
end
```

---

## Step 435: OpenTelemetry Tracing

```elixir
# mix.exs
{:opentelemetry, "~> 1.3"},
{:opentelemetry_api, "~> 1.2"},
{:opentelemetry_exporter, "~> 1.6"},
{:opentelemetry_phoenix, "~> 1.2"},
{:opentelemetry_ecto, "~> 1.2"}

# config/config.exs
config :opentelemetry,
  span_processor: :batch,
  traces_exporter: :otlp

config :opentelemetry_exporter,
  otlp_protocol: :http_protobuf,
  otlp_endpoint: "http://jaeger:4318"

# application.ex
def start(_type, _args) do
  OpentelemetryPhoenix.setup()
  OpentelemetryEcto.setup([:my_app, :repo])
  ...
end
```

```elixir
defmodule MyApp.Orders do
  require OpenTelemetry.Tracer, as: Tracer
  
  def create_order(attrs) do
    Tracer.with_span "orders.create" do
      Tracer.set_attributes([
        {"user.id", attrs[:user_id]},
        {"order.items.count", length(attrs[:items])}
      ])
      
      Ecto.Multi.new()
      |> Ecto.Multi.insert(:order, Order.changeset(%Order{}, attrs))
      |> Ecto.Multi.run(:inventory, fn _repo, %{order: order} ->
        Tracer.with_span "inventory.reserve" do
          reserve_inventory(order.items)
        end
      end)
      |> Ecto.Multi.run(:payment, fn _repo, %{order: order} ->
        Tracer.with_span "payment.charge" do
          Tracer.set_attribute("payment.amount", order.total)
          charge_payment(order)
        end
      end)
      |> Repo.transaction()
    end
  end
end
```

---

## Step 436: Structured Logging

```elixir
# mix.exs
{:logger_json, "~> 5.1"}

# config/config.exs
config :logger,
  backends: [LoggerJSON]

config :logger_json, :backend,
  formatter: LoggerJSON.Formatters.GoogleCloud,
  metadata: [:request_id, :user_id, :trace_id]

# Contextual logging
defmodule MyApp.Logger do
  require Logger
  
  def with_context(metadata, fun) do
    Logger.metadata(metadata)
    result = fun.()
    Logger.reset_metadata()
    result
  end
  
  def log_request(conn, duration) do
    Logger.info("HTTP request",
      method:        conn.method,
      path:          conn.request_path,
      status:        conn.status,
      duration_ms:   System.convert_time_unit(duration, :native, :millisecond),
      ip:            conn.remote_ip |> :inet.ntoa() |> to_string(),
      user_agent:    get_req_header(conn, "user-agent") |> List.first(),
      request_id:    conn.assigns[:request_id],
      user_id:       conn.assigns[:current_user] && conn.assigns.current_user.id
    )
  end
end

# Plug for request logging
defmodule MyAppWeb.Plugs.RequestLogger do
  import Plug.Conn
  
  def init(opts), do: opts
  
  def call(conn, _opts) do
    start = System.monotonic_time()
    
    register_before_send(conn, fn conn ->
      duration = System.monotonic_time() - start
      MyApp.Logger.log_request(conn, duration)
      conn
    end)
  end
end
```

---

## Step 437: Log Aggregation (Loki)

```yaml
# docker-compose.yml
loki:
  image: grafana/loki:2.9.0
  ports:
    - "3100:3100"
  volumes:
    - ./loki-config.yml:/etc/loki/local-config.yaml

promtail:
  image: grafana/promtail:2.9.0
  volumes:
    - /var/log:/var/log
    - ./promtail-config.yml:/etc/promtail/config.yml
```

```yaml
# promtail-config.yml
server:
  http_listen_port: 9080

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: elixir_app
    static_configs:
      - targets:
          - localhost
        labels:
          job: elixir_app
          __path__: /var/log/elixir/*.log
    pipeline_stages:
      - json:
          expressions:
            level: level
            message: message
            request_id: request_id
      - labels:
          level:
          request_id:
```

---

## Step 438: Error Tracking (Sentry)

```elixir
# mix.exs
{:sentry, "~> 10.0"},
{:hackney, "~> 1.8"}  # HTTP client for Sentry

# config/config.exs
config :sentry,
  dsn: System.get_env("SENTRY_DSN"),
  environment_name: Mix.env(),
  enable_source_code_context: true,
  root_source_code_paths: [File.cwd!()],
  tags: %{app: "my_app"}

# endpoint.ex
plug Sentry.PlugCapture

# error_view.ex
defmodule MyAppWeb.ErrorView do
  def render("500.html", %{conn: conn}) do
    Sentry.capture_exception(conn.assigns[:reason] || RuntimeError.exception("Unknown error"),
      stacktrace: conn.assigns[:stack],
      extra: %{
        request_id: conn.assigns[:request_id],
        user_id:    conn.assigns[:current_user] && conn.assigns.current_user.id
      }
    )
    
    "Internal server error"
  end
end

# Manual capture
defmodule MyApp.ErrorReporter do
  def capture(exception, opts \\ []) do
    Sentry.capture_exception(exception,
      [stacktrace: __STACKTRACE__] ++ opts
    )
  end
  
  def capture_message(message, opts \\ []) do
    Sentry.capture_message(message, opts)
  end
end
```

---

## Step 439: APM Dashboard

```elixir
# LiveView dashboard for real-time monitoring
defmodule MyAppWeb.DashboardLive do
  use MyAppWeb, :live_view
  
  @refresh_interval 5_000
  
  def mount(_params, _session, socket) do
    if connected?(socket) do
      :timer.send_interval(@refresh_interval, :refresh)
    end
    
    {:ok, assign(socket, metrics: get_metrics())}
  end
  
  def handle_info(:refresh, socket) do
    {:noreply, assign(socket, metrics: get_metrics())}
  end
  
  defp get_metrics do
    %{
      request_rate:      get_request_rate(),
      error_rate:        get_error_rate(),
      p50_latency:       get_latency(:p50),
      p99_latency:       get_latency(:p99),
      active_users:      get_active_users(),
      db_query_rate:     get_db_query_rate(),
      memory_mb:         get_memory_mb(),
      process_count:     :erlang.system_info(:process_count),
      scheduler_usage:   get_scheduler_usage()
    }
  end
  
  defp get_memory_mb do
    :erlang.memory(:total) |> div(1_048_576)
  end
  
  defp get_scheduler_usage do
    :scheduler.utilization(1)
    |> Enum.map(fn {_, util, _} -> util end)
    |> Enum.sum()
    |> then(&(&1 / :erlang.system_info(:schedulers)))
    |> Float.round(2)
  end
  
  def render(assigns) do
    ~H"""
    <div class="monitoring-dashboard">
      <.metric label="Request Rate" value={"#{@metrics.request_rate}/s"} />
      <.metric label="Error Rate" value={"#{@metrics.error_rate}%"} />
      <.metric label="P50 Latency" value={"#{@metrics.p50_latency}ms"} />
      <.metric label="P99 Latency" value={"#{@metrics.p99_latency}ms"} />
      <.metric label="Memory" value={"#{@metrics.memory_mb}MB"} />
      <.metric label="Processes" value={@metrics.process_count} />
    </div>
    """
  end
end
```

---

## Step 440: Alerting

```elixir
defmodule MyApp.Alerting do
  @thresholds %{
    error_rate:    5.0,    # %
    p99_latency:   1000,   # ms
    memory_mb:     2000,
    queue_depth:   1000
  }
  
  def check_alerts do
    metrics = get_current_metrics()
    
    @thresholds
    |> Enum.each(fn {metric, threshold} ->
      value = Map.get(metrics, metric)
      
      if value && value > threshold do
        fire_alert(metric, value, threshold)
      end
    end)
  end
  
  defp fire_alert(metric, value, threshold) do
    message = "[ALERT] #{metric} = #{value} (threshold: #{threshold})"
    
    # Send to PagerDuty, Slack, email, etc.
    send_to_slack(message)
    send_to_pagerduty(message) if :critical in get_severity(metric)
    
    :telemetry.execute([:my_app, :alert, :fired],
      %{value: value},
      %{metric: metric, threshold: threshold}
    )
  end
  
  defp send_to_slack(message) do
    webhook_url = System.get_env("SLACK_WEBHOOK_URL")
    
    Finch.build(:post, webhook_url, [{"content-type", "application/json"}],
      Jason.encode!(%{text: message})
    )
    |> Finch.request(MyApp.Finch)
  end
end

# Schedule alert checks via Oban
defmodule MyApp.Workers.AlertChecker do
  use Oban.Worker, queue: :monitoring
  
  @impl Oban.Worker
  def perform(_job) do
    MyApp.Alerting.check_alerts()
    :ok
  end
end
```

---

## สรุป Part 39

✅ **Step 431** - Observability concepts  
✅ **Step 432** - Telemetry setup  
✅ **Step 433** - Custom metrics  
✅ **Step 434** - Prometheus + Grafana  
✅ **Step 435** - OpenTelemetry tracing  
✅ **Step 436** - Structured logging  
✅ **Step 437** - Loki log aggregation  
✅ **Step 438** - Sentry error tracking  
✅ **Step 439** - APM dashboard  
✅ **Step 440** - Alerting  

➡️ [Part 40: Testing Mastery](./part-40-testing.md)
