# Part 72: Observability & Monitoring (Steps 781-800)

## Step 781: Observability Pillars

```
The Three Pillars of Observability:

┌─────────────────────────────────────────────────────┐
│  METRICS (what's happening?)                        │
│  - Request rate, error rate, latency (RED metrics)  │
│  - CPU, memory, DB connections (USE metrics)        │
│  - Business metrics (revenue, signups, conversions) │
│  Tools: Prometheus, StatsD, :telemetry              │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  LOGS (what happened?)                              │
│  - Structured JSON logs                             │
│  - Request/response details                         │
│  - Errors with stack traces                         │
│  Tools: Logger, Loki, Elasticsearch                 │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  TRACES (why did it happen?)                        │
│  - Distributed request tracing                      │
│  - Spans across services                            │
│  - Performance bottleneck identification            │
│  Tools: OpenTelemetry, Jaeger, Honeycomb            │
└─────────────────────────────────────────────────────┘
```

---

## Step 782: Telemetry Events

```elixir
# :telemetry is the core Elixir observability library
# All major libraries (Phoenix, Ecto, Oban) emit telemetry events

# Emit custom telemetry
defmodule MyApp.Orders do
  def create_order(attrs) do
    start = System.monotonic_time()

    result = do_create_order(attrs)

    :telemetry.execute(
      [:my_app, :orders, :created],
      %{
        duration: System.monotonic_time() - start,
        count: 1
      },
      %{
        status: if(match?({:ok, _}, result), do: :success, else: :error),
        plan:   attrs[:plan]
      }
    )

    result
  end

  # Span-style measurement
  def fulfill_order(order_id) do
    :telemetry.span(
      [:my_app, :orders, :fulfillment],
      %{order_id: order_id},
      fn ->
        result = do_fulfill(order_id)
        {result, %{}}
      end
    )
  end
end

# Attach handlers
defmodule MyApp.Telemetry.Handlers do
  def attach_all do
    :telemetry.attach_many("my-app-handlers", [
      [:my_app, :orders, :created],
      [:phoenix, :router_dispatch, :stop],
      [:ecto, :repo, :query]
    ], &handle_event/4, nil)
  end

  def handle_event([:my_app, :orders, :created], measurements, metadata, _config) do
    :prometheus_counter.inc(:order_created_total, [metadata.status, metadata.plan])
    :prometheus_histogram.observe(:order_creation_duration_ms,
      System.convert_time_unit(measurements.duration, :native, :millisecond)
    )
  end

  def handle_event([:phoenix, :router_dispatch, :stop], measurements, metadata, _config) do
    duration_ms = System.convert_time_unit(measurements.duration, :native, :millisecond)
    
    :prometheus_histogram.observe(:http_request_duration_ms,
      [metadata.method, to_string(metadata.status)],
      duration_ms
    )
  end
end
```

---

## Step 783: Prometheus Integration

```elixir
# mix.exs: {:prometheus_ex, "~> 3.0"}, {:prometheus_plugs, "~> 1.1"}

defmodule MyApp.Metrics do
  use Prometheus.Metric

  def setup do
    Counter.declare(
      name:   :http_requests_total,
      labels: [:method, :path, :status],
      help:   "Total HTTP requests"
    )

    Histogram.declare(
      name:    :http_request_duration_ms,
      labels:  [:method, :status],
      buckets: [5, 10, 25, 50, 100, 250, 500, 1000],
      help:    "HTTP request duration in milliseconds"
    )

    Gauge.declare(
      name:   :active_connections,
      labels: [],
      help:   "Current active connections"
    )

    Counter.declare(
      name:   :order_created_total,
      labels: [:status, :plan],
      help:   "Total orders created"
    )

    Gauge.declare(
      name:   :queue_size,
      labels: [:queue],
      help:   "Oban queue size"
    )
  end
end

# Expose /metrics endpoint
defmodule MyAppWeb.MetricsController do
  use MyAppWeb, :controller

  def index(conn, _params) do
    metrics = :prometheus_text_format.format()
    
    conn
    |> put_resp_content_type("text/plain; version=0.0.4")
    |> send_resp(200, metrics)
  end
end

# Router (restrict to internal network or add auth)
scope "/internal" do
  get "/metrics", MetricsController, :index
end
```

---

## Step 784: Structured Logging

```elixir
defmodule MyApp.Logger do
  require Logger

  # Add structured metadata to all logs
  def request_context(conn) do
    Logger.metadata(
      request_id:  conn.assigns[:request_id] || generate_id(),
      user_id:     conn.assigns[:current_user]&.id,
      tenant_id:   conn.assigns[:current_tenant]&.id,
      remote_ip:   conn.remote_ip |> Tuple.to_list() |> Enum.join(".")
    )
  end

  def info(message, metadata \\ []) do
    Logger.info(fn ->
      %{
        message:   message,
        timestamp: DateTime.utc_now(),
        level:     "info"
      }
      |> Map.merge(Map.new(metadata))
      |> Jason.encode!()
    end)
  end

  def error(message, metadata \\ []) do
    Logger.error(fn ->
      %{
        message: message,
        level:   "error"
      }
      |> Map.merge(Map.new(metadata))
      |> Jason.encode!()
    end)
  end

  defp generate_id, do: :crypto.strong_rand_bytes(8) |> Base.encode16(case: :lower)
end

# config.exs - use JSON formatter
config :logger, :console,
  format:   "$message\n",
  metadata: [:request_id, :user_id, :tenant_id]

# Or use LoggerJSON library
config :logger,
  backends: [LoggerJSON]

config :logger_json, :backend,
  formatter: LoggerJSON.Formatters.BasicLogger
```

---

## Step 785: OpenTelemetry Distributed Tracing

```elixir
# mix.exs
{:opentelemetry, "~> 1.3"},
{:opentelemetry_api, "~> 1.2"},
{:opentelemetry_exporter, "~> 1.6"},
{:opentelemetry_phoenix, "~> 1.2"},
{:opentelemetry_ecto, "~> 1.1"}

# config.exs
config :opentelemetry, :resource,
  service: [name: "my-app", version: "1.0.0"]

config :opentelemetry_exporter,
  otlp_protocol: :http_protobuf,
  otlp_endpoint: "https://api.honeycomb.io",
  otlp_headers:  [{"x-honeycomb-team", System.get_env("HONEYCOMB_API_KEY")}]

# application.ex
def start(_type, _args) do
  OpentelemetryPhoenix.setup()
  OpentelemetryEcto.setup([:my_app, :repo])
  
  # ...
end

# Custom spans
defmodule MyApp.PaymentService do
  require OpenTelemetry.Tracer

  def charge(user, amount) do
    OpenTelemetry.Tracer.with_span "payment.charge" do
      OpenTelemetry.Tracer.set_attribute("user.id", user.id)
      OpenTelemetry.Tracer.set_attribute("payment.amount", amount)

      case Stripe.Charge.create(%{amount: amount, customer: user.stripe_id}) do
        {:ok, charge} ->
          OpenTelemetry.Tracer.set_attribute("payment.stripe_id", charge.id)
          {:ok, charge}
        {:error, error} ->
          OpenTelemetry.Tracer.set_status(:error, error.message)
          {:error, error}
      end
    end
  end
end
```

---

## Step 786: Alerting

```elixir
defmodule MyApp.Alerting do
  # Threshold-based alerts via PagerDuty / Slack

  def check_error_rate do
    rate = MyApp.Metrics.error_rate_last_5min()
    
    if rate > 0.05 do
      alert(:high_error_rate, %{
        rate:      rate,
        threshold: 0.05,
        message:   "Error rate #{Float.round(rate * 100, 1)}% exceeds 5% threshold"
      })
    end
  end

  def check_response_time do
    p99 = MyApp.Metrics.p99_response_time()
    
    if p99 > 2000 do  # 2 seconds
      alert(:slow_responses, %{
        p99:       p99,
        threshold: 2000,
        message:   "P99 response time #{p99}ms exceeds 2000ms threshold"
      })
    end
  end

  defp alert(type, metadata) do
    Logger.error("ALERT [#{type}]: #{metadata.message}", metadata: metadata)
    
    # Send to Slack
    Task.start(fn ->
      Req.post!(
        System.fetch_env!("SLACK_WEBHOOK_URL"),
        json: %{
          text: ":rotating_light: *#{type}*\n#{metadata.message}",
          attachments: [%{
            color: "danger",
            fields: Enum.map(metadata, fn {k, v} ->
              %{title: to_string(k), value: to_string(v), short: true}
            end)
          }]
        }
      )
    end)
  end
end
```

---

## Step 787: Performance Profiling

```elixir
defmodule MyApp.Profiler do
  # Profile with :eprof, :cprof, or :fprof

  def profile_function(fun) do
    :eprof.start_profiling([self()])
    result = fun.()
    :eprof.stop_profiling()
    :eprof.analyze(:total)
    result
  end

  def measure(name, fun) do
    start  = System.monotonic_time(:microsecond)
    result = fun.()
    elapsed = System.monotonic_time(:microsecond) - start
    
    Logger.debug("#{name} took #{elapsed}µs")
    
    {result, elapsed}
  end

  # Memory profiling
  def measure_memory(fun) do
    :erlang.garbage_collect()
    {memory_before, _} = :erlang.process_info(self(), :memory)
    
    result = fun.()
    
    :erlang.garbage_collect()
    {memory_after, _} = :erlang.process_info(self(), :memory)
    
    diff = memory_after - memory_before
    Logger.debug("Memory delta: #{diff} bytes")
    
    {result, diff}
  end

  # Flamegraph with eflambe
  def flamegraph(module, function, args) do
    :eflambe.apply({module, function, args}, [output_format: :brendan_gregg])
  end
end
```

---

## Step 788: LiveDashboard

```elixir
# mix.exs: {:phoenix_live_dashboard, "~> 0.8"}

# router.ex
import Phoenix.LiveDashboard.Router

if Mix.env() in [:dev, :staging] do
  scope "/" do
    pipe_through [:browser, :require_admin]
    live_dashboard "/dashboard",
      metrics: MyAppWeb.Telemetry,
      ecto_repos: [MyApp.Repo]
  end
end

# telemetry.ex
defmodule MyAppWeb.Telemetry do
  use Supervisor
  import Telemetry.Metrics

  def start_link(arg) do
    Supervisor.start_link(__MODULE__, arg, name: __MODULE__)
  end

  def init(_arg) do
    children = [
      {Telemetry.Metrics.ConsoleReporter, metrics: metrics()}
    ]
    Supervisor.init(children, strategy: :one_for_one)
  end

  def metrics do
    [
      # Phoenix
      summary("phoenix.endpoint.stop.duration",
        unit: {:native, :millisecond}),
      counter("phoenix.router_dispatch.stop.duration",
        tags: [:route]),

      # Ecto
      summary("my_app.repo.query.total_time",
        unit: {:native, :millisecond},
        description: "The sum of the other measurements"),

      # VM
      summary("vm.memory.total", unit: {:byte, :kilobyte}),
      summary("vm.total_run_queue_lengths.total"),
      summary("vm.total_run_queue_lengths.cpu"),

      # Custom
      counter("my_app.orders.created.count",
        tags: [:status]),
    ]
  end
end
```

---

## Step 789: Error Tracking

```elixir
# mix.exs: {:sentry, "~> 10.0"}

# config.exs
config :sentry,
  dsn:              System.get_env("SENTRY_DSN"),
  environment_name: Mix.env(),
  enable_source_code_context: true,
  root_source_code_paths: [File.cwd!()]

# Capture exceptions
def handle_error(reason) do
  Sentry.capture_exception(reason,
    extra: %{
      user_id: current_user_id(),
      context: "payment_processing"
    },
    tags: %{
      service: "payments",
      environment: "production"
    }
  )
end

# Custom error handler
defmodule MyApp.ErrorHandler do
  def handle_event([:my_app, :error], _measurements, metadata, _config) do
    Sentry.capture_message(metadata.message,
      level: "error",
      extra: metadata
    )
  end
end

# Global error capture in GenServers
defmodule MyApp.SafeWorker do
  def safe_perform(fun) do
    try do
      fun.()
    rescue
      error ->
        Sentry.capture_exception(error, stacktrace: __STACKTRACE__)
        {:error, error}
    end
  end
end
```

---

## Step 790: SLO/SLA Monitoring

```elixir
defmodule MyApp.SLO do
  # Service Level Objectives monitoring

  @slos %{
    api_availability:    {99.9, :percentage},
    api_p99_latency:     {500,  :millisecond},
    api_p95_latency:     {200,  :millisecond},
    order_success_rate:  {99.5, :percentage},
    email_delivery_rate: {98.0, :percentage}
  }

  def check_all do
    @slos
    |> Enum.map(fn {name, {target, unit}} ->
      actual = measure(name)
      status = if within_slo?(actual, target, unit), do: :ok, else: :breached

      %{
        name:   name,
        target: target,
        actual: actual,
        unit:   unit,
        status: status
      }
    end)
  end

  def error_budget(slo_name) do
    {target, _unit} = Map.fetch!(@slos, slo_name)
    actual = measure(slo_name)
    
    budget_allowed  = 100.0 - target
    budget_consumed = max(0, 100.0 - actual)
    
    %{
      total_allowed:  budget_allowed,
      consumed:       budget_consumed,
      remaining:      budget_allowed - budget_consumed,
      percent_used:   budget_consumed / budget_allowed * 100
    }
  end

  defp measure(:api_availability) do
    # Calculate from logs/metrics
    total   = MyApp.Metrics.total_requests_last_30d()
    errors  = MyApp.Metrics.error_requests_last_30d()
    (total - errors) / total * 100
  end

  defp within_slo?(actual, target, :percentage), do: actual >= target
  defp within_slo?(actual, target, :millisecond), do: actual <= target
end
```

---

## สรุป Part 72

✅ **Step 781** - Observability pillars  
✅ **Step 782** - Telemetry events  
✅ **Step 783** - Prometheus integration  
✅ **Step 784** - Structured logging  
✅ **Step 785** - OpenTelemetry tracing  
✅ **Step 786** - Alerting  
✅ **Step 787** - Performance profiling  
✅ **Step 788** - LiveDashboard  
✅ **Step 789** - Error tracking  
✅ **Step 790** - SLO/SLA monitoring  

➡️ [Part 73: GraphQL with Absinthe](./part-73-graphql.md)
