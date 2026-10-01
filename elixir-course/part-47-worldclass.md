# Part 47: World-Class Patterns (Steps 521-560)

## Step 521: Zero-Downtime Deploy

```elixir
defmodule MyApp.Release do
  def migrate do
    {:ok, _} = Application.ensure_all_started(:ecto_sql)
    
    for repo <- repos() do
      {:ok, _, _} = Ecto.Migrator.with_repo(repo, &Ecto.Migrator.run(&1, :up, all: true))
    end
  end
  
  def rollback(repo, version) do
    {:ok, _, _} = Ecto.Migrator.with_repo(repo, &Ecto.Migrator.run(&1, :down, to: version))
  end
  
  defp repos do
    Application.fetch_env!(:my_app, :ecto_repos)
  end
end

# Hot code reload (Erlang capability)
defmodule MyApp.HotReload do
  def reload_module(module) do
    :code.purge(module)
    :code.load_file(module)
    {:ok, module}
  end
  
  # Soft upgrade (state migration across code change)
  def upgrade(old_vsn, extra) do
    :release_handler.install_release("2.0.0")
    :release_handler.make_permanent("2.0.0")
  end
end

# Rolling deploy strategy
# Kubernetes: maxSurge: 1, maxUnavailable: 0
# Each new pod runs: bin/app eval "MyApp.Release.migrate()"
# before receiving traffic (readiness probe)
```

---

## Step 522: Feature Flags

```elixir
defmodule MyApp.FeatureFlags do
  @flags_table :feature_flags
  
  def setup do
    :ets.new(@flags_table, [:set, :public, :named_table, read_concurrency: true])
    load_from_config()
  end
  
  def enabled?(flag, context \\ %{}) do
    case :ets.lookup(@flags_table, flag) do
      [{^flag, :always}]   -> true
      [{^flag, :never}]    -> false
      [{^flag, {:rollout, pct}}] -> rollout_enabled?(context, pct)
      [{^flag, {:user_segment, segment}}] -> context[:segment] == segment
      [] -> false
    end
  end
  
  def enable(flag), do: :ets.insert(@flags_table, {flag, :always})
  def disable(flag), do: :ets.insert(@flags_table, {flag, :never})
  
  def set_rollout(flag, percentage) when percentage in 0..100 do
    :ets.insert(@flags_table, {flag, {:rollout, percentage}})
  end
  
  defp rollout_enabled?(%{user_id: user_id}, percentage) do
    hash = :erlang.phash2(user_id, 100)
    hash < percentage
  end
  defp rollout_enabled?(_, _), do: false
  
  defp load_from_config do
    flags = Application.get_env(:my_app, :feature_flags, [])
    Enum.each(flags, fn {flag, value} ->
      :ets.insert(@flags_table, {flag, value})
    end)
  end
end

# Usage
if MyApp.FeatureFlags.enabled?(:new_checkout, %{user_id: conn.assigns.current_user.id}) do
  # new flow
else
  # old flow
end
```

---

## Step 523: A/B Testing

```elixir
defmodule MyApp.ABTest do
  def variant(experiment, user_id) do
    bucket = :erlang.phash2("#{experiment}:#{user_id}", 100)
    
    variants = get_variants(experiment)
    
    Enum.reduce_while(variants, nil, fn {variant, weight, acc_weight}, _result ->
      new_acc = acc_weight + weight
      if bucket < new_acc do
        {:halt, variant}
      else
        {:cont, new_acc}
      end
    end)
  end
  
  def track_conversion(experiment, user_id, variant, value \\ 1) do
    :telemetry.execute(
      [:ab_test, :conversion],
      %{value: value},
      %{experiment: experiment, variant: variant, user_id: user_id}
    )
  end
  
  defp get_variants(experiment) do
    Application.get_env(:my_app, :ab_tests, %{})
    |> Map.get(experiment, [{"control", 50, 0}, {"treatment", 50, 50}])
  end
end

# config/config.exs
config :my_app, :ab_tests, %{
  "checkout_button_color" => [
    {"blue",   50, 0},
    {"green",  30, 50},
    {"orange", 20, 80}
  ],
  "new_recommendation_algo" => [
    {"control",   60, 0},
    {"treatment", 40, 60}
  ]
}
```

---

## Step 524: Chaos Engineering

```elixir
defmodule MyApp.ChaosMonkey do
  @moduledoc "Chaos engineering for resilience testing"
  
  def inject_latency(fun, opts \\ []) do
    if chaos_enabled?() do
      delay = Keyword.get(opts, :max_ms, 500) |> :rand.uniform()
      Process.sleep(delay)
    end
    fun.()
  end
  
  def inject_failure(fun, failure_rate \\ 0.1) do
    if chaos_enabled?() and :rand.uniform() < failure_rate do
      raise "Chaos monkey injection"
    end
    fun.()
  end
  
  def kill_random_process do
    if chaos_enabled?() do
      processes = Process.list()
      victim    = Enum.random(processes)
      # Don't kill critical processes
      unless victim in critical_processes() do
        Process.exit(victim, :kill)
      end
    end
  end
  
  def simulate_network_partition do
    if chaos_enabled?() do
      nodes = Node.list()
      victim = Enum.random(nodes)
      Node.disconnect(victim)
      
      Process.send_after(self(), {:reconnect_node, victim}, 10_000)
    end
  end
  
  defp chaos_enabled? do
    Application.get_env(:my_app, :chaos_enabled, false) and
    Mix.env() != :prod
  end
  
  defp critical_processes do
    [Process.whereis(:kernel_safe_sup),
     Process.whereis(:application_controller)]
    |> Enum.reject(&is_nil/1)
  end
end
```

---

## Step 525: CQRS Advanced

```elixir
defmodule MyApp.CQRS do
  # Command Bus
  defmodule CommandBus do
    def dispatch(%{__struct__: command_type} = command) do
      handler = get_handler(command_type)
      
      :telemetry.span([:cqrs, :command], %{command: command_type}, fn ->
        result = handler.handle(command)
        {result, %{result: if(match?({:ok, _}, result), do: :ok, else: :error)}}
      end)
    end
    
    defp get_handler(command_type) do
      handlers = Application.get_env(:my_app, :command_handlers, %{})
      
      case Map.get(handlers, command_type) do
        nil     -> raise "No handler for #{command_type}"
        handler -> handler
      end
    end
  end
  
  # Query Bus
  defmodule QueryBus do
    def execute(%{__struct__: query_type} = query) do
      handler = get_handler(query_type)
      handler.execute(query)
    end
    
    defp get_handler(query_type) do
      handlers = Application.get_env(:my_app, :query_handlers, %{})
      Map.get(handlers, query_type) || raise "No handler for #{query_type}"
    end
  end
end

# Commands
defmodule Commands.PlaceOrder do
  defstruct [:user_id, :items, :payment_method]
end

defmodule Commands.CancelOrder do
  defstruct [:order_id, :reason]
end

# Command handlers
defmodule Handlers.PlaceOrderHandler do
  def handle(%Commands.PlaceOrder{} = cmd) do
    Orders.create_order(cmd.user_id, cmd.items, cmd.payment_method)
  end
end

# Queries
defmodule Queries.GetOrderSummary do
  defstruct [:user_id, :page, :limit]
end

# Query handlers
defmodule Handlers.GetOrderSummaryHandler do
  def execute(%Queries.GetOrderSummary{} = q) do
    Orders.ReadModel.list_for_user(q.user_id, page: q.page, limit: q.limit)
  end
end
```

---

## Step 526: DDD (Domain Driven Design)

```elixir
# Aggregate
defmodule Orders.Aggregate do
  defstruct [:id, :user_id, :items, :status, :version]
  
  def new(id, user_id) do
    %__MODULE__{id: id, user_id: user_id, items: [], status: :new, version: 0}
  end
  
  def add_item(%{status: :new} = order, product_id, qty, price) do
    {:ok, %{order |
      items: [%{product_id: product_id, quantity: qty, price: price} | order.items],
      version: order.version + 1
    }}
  end
  def add_item(_, _, _, _), do: {:error, :cannot_modify}
  
  def submit(%{status: :new, items: [_ | _]} = order) do
    {:ok, %{order | status: :submitted, version: order.version + 1}}
  end
  def submit(%{items: []}), do: {:error, :empty_order}
  def submit(_), do: {:error, :already_submitted}
  
  def total(order) do
    Enum.reduce(order.items, Decimal.new("0"), fn item, acc ->
      Decimal.add(acc, Decimal.mult(item.price, Decimal.new(item.quantity)))
    end)
  end
end

# Value Object
defmodule Orders.Money do
  @enforce_keys [:amount, :currency]
  defstruct [:amount, :currency]
  
  def new(amount, currency \\ "THB") when is_binary(currency) do
    case Decimal.parse(to_string(amount)) do
      {d, ""} when Decimal.positive?(d) -> {:ok, %__MODULE__{amount: d, currency: currency}}
      _ -> {:error, :invalid_amount}
    end
  end
  
  def add(%__MODULE__{currency: c} = a, %__MODULE__{currency: c} = b) do
    {:ok, %{a | amount: Decimal.add(a.amount, b.amount)}}
  end
  def add(_, _), do: {:error, :currency_mismatch}
  
  defimpl String.Chars do
    def to_string(%{amount: a, currency: c}), do: "#{a} #{c}"
  end
end
```

---

## Step 527: Hexagonal Architecture

```
Hexagonal / Ports and Adapters:

  ┌──────────────────────────────────────┐
  │                                      │
  │   ┌──────────────────────────────┐   │
  │   │        Domain Core           │   │
  │   │  (pure functions, no deps)   │   │
  │   └────────────┬─────────────────┘   │
  │                │ Ports (behaviours)  │
  │   ┌────────────▼─────────────────┐   │
  │   │          Adapters            │   │
  │   │  (HTTP, DB, Email, Queue)    │   │
  │   └──────────────────────────────┘   │
  │                                      │
  └──────────────────────────────────────┘
```

```elixir
# Port (behaviour)
defmodule Ports.OrderRepository do
  @callback save(order :: map()) :: {:ok, map()} | {:error, term()}
  @callback get(id :: String.t()) :: {:ok, map()} | {:error, :not_found}
  @callback list_for_user(user_id :: integer()) :: [map()]
end

# Adapter: PostgreSQL
defmodule Adapters.PostgresOrderRepository do
  @behaviour Ports.OrderRepository
  
  alias MyApp.Repo
  alias MyApp.Orders.Order
  
  @impl Ports.OrderRepository
  def save(order) do
    case Repo.insert(Order.changeset(%Order{}, order)) do
      {:ok, record} -> {:ok, to_domain(record)}
      {:error, cs}  -> {:error, cs}
    end
  end
  
  @impl Ports.OrderRepository
  def get(id) do
    case Repo.get(Order, id) do
      nil  -> {:error, :not_found}
      rec  -> {:ok, to_domain(rec)}
    end
  end
  
  @impl Ports.OrderRepository
  def list_for_user(user_id) do
    import Ecto.Query
    from(o in Order, where: o.user_id == ^user_id)
    |> Repo.all()
    |> Enum.map(&to_domain/1)
  end
  
  defp to_domain(record), do: Map.from_struct(record)
end

# Domain service (pure, no dependencies)
defmodule Domain.OrderService do
  def create_order(repo, user_id, items) do
    with {:ok, validated_items} <- validate_items(items),
         total <- calculate_total(validated_items),
         order <- build_order(user_id, validated_items, total),
         {:ok, saved} <- repo.save(order) do
      {:ok, saved}
    end
  end
  
  defp validate_items([]), do: {:error, :empty_order}
  defp validate_items(items), do: {:ok, items}
  
  defp calculate_total(items) do
    Enum.reduce(items, Decimal.new("0"), fn i, acc ->
      Decimal.add(acc, Decimal.mult(i.price, Decimal.new(i.quantity)))
    end)
  end
  
  defp build_order(user_id, items, total) do
    %{id: Ecto.UUID.generate(), user_id: user_id, items: items, total: total, status: :pending}
  end
end
```

---

## Step 528: Performance Profiling

```elixir
# Profiling tools
defmodule MyApp.Profiler do
  # eprof - time profiling
  def profile_time(fun) do
    :eprof.start()
    :eprof.start_profiling([self()])
    
    result = fun.()
    
    :eprof.stop_profiling()
    :eprof.analyze(:total)
    :eprof.stop()
    
    result
  end
  
  # fprof - more detailed call graph
  def profile_calls(fun) do
    :fprof.start()
    :fprof.apply(fun, [])
    :fprof.profile()
    :fprof.analyse(totals: true, details: true, callers: true)
  end
  
  # tprof (OTP 26+) - per-process profiling
  def profile_process(pid, duration_ms \\ 5_000) do
    :tprof.start_link(%{type: :call_count})
    :tprof.enable(pid)
    Process.sleep(duration_ms)
    :tprof.disable(pid)
    :tprof.collect(pid)
    |> :tprof.format()
  end
  
  # Benchmark with Benchee
  def benchmark(scenarios) do
    Benchee.run(scenarios,
      time:   5,
      warmup: 2,
      formatters: [
        Benchee.Formatters.Console,
        {Benchee.Formatters.HTML, file: "bench/output.html"}
      ]
    )
  end
end

# Benchmarking example
MyApp.Profiler.benchmark(%{
  "ETS lookup"    => fn -> :ets.lookup(:cache, :key) end,
  "Process dict"  => fn -> Process.get(:key) end,
  "Map lookup"    => fn -> Map.get(%{key: :val}, :key) end
})
```

---

## Step 529: Telemetry Spans

```elixir
defmodule MyApp.Telemetry.Spans do
  def span(event, metadata \\ %{}, fun) do
    start_time = System.monotonic_time()
    
    :telemetry.execute(event ++ [:start], %{system_time: System.system_time()}, metadata)
    
    try do
      result = fun.()
      
      :telemetry.execute(event ++ [:stop],
        %{duration: System.monotonic_time() - start_time},
        Map.put(metadata, :result, :ok)
      )
      
      result
    rescue
      exception ->
        :telemetry.execute(event ++ [:exception],
          %{duration: System.monotonic_time() - start_time},
          Map.merge(metadata, %{kind: :error, reason: exception, stacktrace: __STACKTRACE__})
        )
        reraise exception, __STACKTRACE__
    end
  end
end

# Attach handlers
defmodule MyApp.Telemetry.Handler do
  def setup do
    events = [
      [:my_app, :repo, :query, :stop],
      [:phoenix, :router_dispatch, :stop],
      [:oban, :job, :stop],
      [:my_app, :orders, :create, :stop]
    ]
    
    :telemetry.attach_many("my-app-handler", events, &handle_event/4, nil)
  end
  
  def handle_event(event, measurements, metadata, _config) do
    duration_ms = measurements[:duration]
      |> then(& if &1, do: System.convert_time_unit(&1, :native, :millisecond))
    
    :logger.info("#{inspect(event)}: #{duration_ms}ms #{inspect(metadata)}")
  end
end
```

---

## Step 530: Production Readiness Checklist

```
Production Readiness Checklist for Elixir Apps:

SECURITY ✅
□ HTTPS enforced (Plug.SSL)
□ HSTS headers
□ CSP headers
□ CSRF protection
□ Input validation (Ecto changesets)
□ SQL injection prevention (Ecto parameterized)
□ XSS prevention (HEEx auto-escape)
□ Secrets from environment (not code)
□ Password hashing (bcrypt rounds >= 12)
□ JWT secrets rotatable
□ Rate limiting (API + auth endpoints)
□ mix deps.audit clean

PERFORMANCE ✅
□ DB indexes for all foreign keys
□ No N+1 queries (preload associations)
□ DB connection pool sized correctly
□ ETS caching for hot data
□ HTTP keep-alive enabled
□ Asset fingerprinting + CDN
□ Gzip compression

RELIABILITY ✅
□ Supervision tree correct
□ Circuit breakers on external calls
□ Retry logic with exponential backoff
□ Database migrations are backward compatible
□ Health check endpoint (/health)
□ Graceful shutdown (SIGTERM handler)

OBSERVABILITY ✅
□ Structured logging (JSON)
□ Metrics exported (Prometheus)
□ Distributed tracing (OpenTelemetry)
□ Error tracking (Sentry)
□ Alerting configured
□ Log retention policy

OPERATIONS ✅
□ Docker multi-stage build
□ Kubernetes health probes
□ CI/CD pipeline
□ Automated rollback
□ Database backup strategy
□ Runbooks documented
```

---

## สรุป Part 47

✅ **Step 521** - Zero-downtime deploy  
✅ **Step 522** - Feature flags  
✅ **Step 523** - A/B testing  
✅ **Step 524** - Chaos engineering  
✅ **Step 525** - CQRS advanced  
✅ **Step 526** - DDD patterns  
✅ **Step 527** - Hexagonal architecture  
✅ **Step 528** - Performance profiling  
✅ **Step 529** - Telemetry spans  
✅ **Step 530** - Production readiness  

➡️ [Part 48: Interview & Career](./part-48-career.md)
