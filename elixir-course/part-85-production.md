# Part 85: Production Readiness (Steps 911-930)

## Step 911: Graceful Shutdown

```elixir
defmodule MyApp.GracefulShutdown do
  # Handle SIGTERM for graceful shutdown

  def setup do
    :os.set_signal(:sigterm, :handle)
    Process.flag(:trap_exit, true)
  end

  def handle_signal(:sigterm) do
    Logger.info("Received SIGTERM, initiating graceful shutdown")
    
    # Stop accepting new requests
    Phoenix.Endpoint.broadcast_from(MyApp.Endpoint, self(), "system", :shutdown, %{})
    
    # Drain in-flight requests (wait up to 30s)
    await_drain(30_000)
    
    # Flush all queues
    Oban.drain_queue(queue: :default)
    
    # Disconnect from database
    MyApp.Repo.stop()
    
    System.stop(0)
  end

  defp await_drain(timeout) do
    start = System.monotonic_time(:millisecond)
    
    Stream.repeatedly(fn -> :timer.sleep(100) end)
    |> Stream.take_while(fn _ ->
      elapsed = System.monotonic_time(:millisecond) - start
      active  = get_active_requests()
      active > 0 and elapsed < timeout
    end)
    |> Stream.run()
  end

  defp get_active_requests do
    case :ranch.procs(MyApp.HTTP, :connections) do
      {:ok, conns} -> length(conns)
      _            -> 0
    end
  end
end

# In application.ex
def start(_type, _args) do
  MyApp.GracefulShutdown.setup()
  # ...
end
```

---

## Step 912: Feature Flags

```elixir
defmodule MyApp.FeatureFlags do
  # Simple feature flag system using ETS + periodic refresh

  @table :feature_flags

  def setup do
    :ets.new(@table, [:named_table, :public, read_concurrency: true])
    reload()
  end

  def enabled?(flag, user \\ nil) do
    case :ets.lookup(@table, flag) do
      [{_, %{enabled: true}}] -> true
      [{_, %{enabled: false}}] -> false
      [{_, %{rollout_percent: pct}}] when is_number(pct) ->
        # Gradual rollout based on user hash
        if user, do: user_in_rollout?(user, flag, pct), else: false
      [] -> false
    end
  end

  def reload do
    flags = MyApp.Repo.all(FeatureFlag) |> Map.new(&{&1.name, &1})
    :ets.insert(@table, Enum.map(flags, fn {k, v} -> {k, v} end))
  end

  defp user_in_rollout?(user, flag, percent) do
    hash = :erlang.phash2("#{flag}:#{user.id}", 100)
    hash < percent
  end
end

# Macro for conditional code execution
defmodule MyApp.Features do
  defmacro when_enabled(flag, do: block) do
    quote do
      if MyApp.FeatureFlags.enabled?(unquote(flag)) do
        unquote(block)
      end
    end
  end
end

# Usage
import MyApp.Features
when_enabled :new_checkout do
  MyApp.NewCheckout.process(order)
end
```

---

## Step 913: Zero-Downtime Deployments

```elixir
defmodule MyApp.ReleaseManager do
  # Mix release tasks for zero-downtime updates

  def hot_upgrade(from_version, to_version) do
    # Generate and apply .appup file
    appup_path = "rel/#{Mix.Project.app()}/ebin/#{Mix.Project.app()}.appup"
    
    case :release_handler.check_install_release(to_version) do
      {:ok, _} ->
        case :release_handler.install_release(to_version) do
          {:ok, _, _} ->
            :release_handler.make_permanent(to_version)
            Logger.info("Hot upgrade complete: #{from_version} -> #{to_version}")
            :ok
          error ->
            Logger.error("Hot upgrade failed: #{inspect(error)}")
            :release_handler.install_release(from_version)
            {:error, :upgrade_failed}
        end
      {:error, reason} ->
        {:error, reason}
    end
  end

  def rolling_restart do
    # Signal all nodes to restart one at a time
    nodes = [node() | Node.list()]
    
    Enum.each(nodes, fn node ->
      Logger.info("Restarting node: #{node}")
      :rpc.call(node, :init, :restart, [])
      # Wait for node to come back up
      wait_for_node(node, 60_000)
    end)
  end

  defp wait_for_node(node, timeout) do
    deadline = System.monotonic_time(:millisecond) + timeout
    
    Stream.repeatedly(fn ->
      :timer.sleep(1000)
      Node.ping(node)
    end)
    |> Stream.take_while(fn
      :pong -> false
      :pang when System.monotonic_time(:millisecond) < deadline -> true
      _ -> false
    end)
    |> Stream.run()
  end
end
```

---

## Step 914: Error Boundaries

```elixir
defmodule MyApp.ErrorBoundary do
  # Catch and report errors without crashing

  defmacro safe_execute(name, do: block) do
    quote do
      try do
        unquote(block)
      rescue
        e ->
          Logger.error("Error in #{unquote(name)}: #{inspect(e)}\n#{Exception.message(e)}")
          Sentry.capture_exception(e, stacktrace: __STACKTRACE__)
          {:error, e}
      catch
        :exit, reason ->
          Logger.error("Exit in #{unquote(name)}: #{inspect(reason)}")
          {:error, {:exit, reason}}
      end
    end
  end

  # Supervisor-level error boundary
  defmodule IsolatedTask do
    def run(fun, opts \\ []) do
      timeout = Keyword.get(opts, :timeout, 30_000)
      
      task = Task.Supervisor.async_nolink(MyApp.TaskSupervisor, fun)
      
      case Task.yield(task, timeout) || Task.shutdown(task) do
        {:ok, result} -> {:ok, result}
        {:exit, reason} -> {:error, {:task_died, reason}}
        nil -> {:error, :timeout}
      end
    end
  end
end

# Usage
import MyApp.ErrorBoundary

safe_execute "payment processing" do
  MyApp.PaymentService.charge(order)
end
```

---

## Step 915: Configuration Management

```elixir
# config/runtime.exs - comprehensive production config

import Config

if config_env() == :prod do
  config :my_app, MyApp.Repo,
    url:       System.fetch_env!("DATABASE_URL"),
    pool_size: System.get_env("DB_POOL_SIZE", "10") |> String.to_integer()

  config :my_app, MyAppWeb.Endpoint,
    secret_key_base: System.fetch_env!("SECRET_KEY_BASE"),
    url: [host: System.fetch_env!("PHX_HOST"), port: 443, scheme: "https"]

  config :my_app, :redis_url, System.fetch_env!("REDIS_URL")
  config :my_app, :s3_bucket, System.fetch_env!("S3_BUCKET")

  config :my_app, :jwt_secret, System.fetch_env!("JWT_SECRET")

  config :sentry,
    dsn:           System.get_env("SENTRY_DSN"),
    environment:   "production",
    release:       System.get_env("RELEASE_VERSION", "unknown")

  config :my_app, :kafka_brokers,
    System.get_env("KAFKA_BROKERS", "localhost:9092")
    |> String.split(",")
    |> Enum.map(fn broker ->
      [host, port] = String.split(broker, ":")
      {host, String.to_integer(port)}
    end)
end
```

---

## Step 916: Secrets Management

```elixir
defmodule MyApp.Secrets do
  # Rotate secrets without restart using config provider

  def get(key) do
    case Application.get_env(:my_app, key) do
      {:system, env_key} -> System.get_env(env_key)
      {:vault, path}     -> fetch_from_vault(path)
      value              -> value
    end
  end

  defp fetch_from_vault(path) do
    vault_url = System.get_env("VAULT_ADDR", "http://localhost:8200")
    token     = System.get_env("VAULT_TOKEN")
    
    case Req.get("#{vault_url}/v1/secret/data/#{path}",
           headers: [{"x-vault-token", token}]) do
      {:ok, %{status: 200, body: %{"data" => %{"data" => data}}}} ->
        data
      _ ->
        raise "Failed to fetch secret from Vault: #{path}"
    end
  end
end

# AWS Secrets Manager
defmodule MyApp.AWSSecrets do
  def fetch(secret_name) do
    case ExAws.SecretsManager.get_secret_value(secret_name) |> ExAws.request() do
      {:ok, %{"SecretString" => secret}} -> Jason.decode!(secret)
      {:error, reason} -> raise "Cannot fetch secret #{secret_name}: #{inspect(reason)}"
    end
  end
end
```

---

## Step 917: Database Connection Management

```elixir
defmodule MyApp.DBConnectionManager do
  # Handle connection pool exhaustion gracefully

  def with_connection(fun) do
    case Ecto.Adapters.SQL.checkout(MyApp.Repo, []) do
      {:ok, conn} ->
        try do
          fun.(conn)
        after
          MyApp.Repo.checkin(conn)
        end
      {:error, :timeout} ->
        Logger.warning("DB connection pool exhausted")
        :telemetry.execute([:db, :pool_timeout], %{count: 1})
        {:error, :db_unavailable}
    end
  end

  def pool_stats do
    pool = MyApp.Repo.get_dynamic_repo()
    
    %{
      size:       DBConnection.pool_size(pool),
      idle:       DBConnection.pool_idle(pool),
      busy:       DBConnection.pool_busy(pool),
      queue_size: DBConnection.pool_queue_size(pool)
    }
  end

  def check_health do
    try do
      MyApp.Repo.query!("SELECT 1")
      :ok
    rescue
      _ -> {:error, :db_unavailable}
    end
  end
end
```

---

## Step 918: Audit Logging

```elixir
defmodule MyApp.AuditLog do
  use Ecto.Schema
  import Ecto.Changeset

  schema "audit_logs" do
    field :user_id,      :integer
    field :action,       :string
    field :resource,     :string
    field :resource_id,  :string
    field :changes,      :map
    field :ip_address,   :string
    field :user_agent,   :string
    field :request_id,   :string
    field :metadata,     :map

    timestamps(updated_at: false)
  end

  def log(attrs) do
    %__MODULE__{}
    |> cast(attrs, [:user_id, :action, :resource, :resource_id, :changes, :ip_address, :user_agent, :request_id, :metadata])
    |> validate_required([:action, :resource])
    |> MyApp.Repo.insert()
  end

  def log_async(attrs) do
    Task.start(fn -> log(attrs) end)
  end
end

# Audit plug for automatic logging
defmodule MyAppWeb.Plugs.AuditLogger do
  def init(opts), do: opts

  def call(conn, resource: resource) do
    register_before_send(conn, fn conn ->
      if conn.status in 200..299 do
        user_id = conn.assigns[:current_user] && conn.assigns.current_user.id
        
        MyApp.AuditLog.log_async(%{
          user_id:    user_id,
          action:     conn.method,
          resource:   resource,
          resource_id: conn.params["id"],
          ip_address:  to_string(:inet.ntoa(conn.remote_ip)),
          user_agent:  get_req_header(conn, "user-agent") |> List.first(),
          request_id:  Logger.metadata()[:request_id]
        })
      end
      conn
    end)
  end
end
```

---

## Step 919: SLO Monitoring

```elixir
defmodule MyApp.SLO do
  # Service Level Objective monitoring

  @slos %{
    api_latency:     {95, 200},   # 95th percentile < 200ms
    error_rate:      {0.1, nil},  # less than 0.1%
    availability:    {99.9, nil}  # 99.9% uptime
  }

  def check_all do
    Enum.map(@slos, fn {slo_name, target} ->
      {slo_name, check(slo_name, target)}
    end)
  end

  def check(:api_latency, {percentile, max_ms}) do
    # Query Prometheus for latency percentile
    p_value = fetch_metric("histogram_quantile(#{percentile/100}, rate(http_request_duration_ms_bucket[5m]))")
    
    %{
      slo:     :api_latency,
      target:  "p#{percentile} < #{max_ms}ms",
      current: p_value,
      status:  (if p_value <= max_ms, do: :ok, else: :breach)
    }
  end

  def error_budget(slo_name, period_hours \\ 24) do
    {target_percent, _} = Map.get(@slos, slo_name)
    
    error_budget_minutes  = period_hours * 60 * (1 - target_percent / 100)
    consumed_minutes      = get_downtime_minutes(slo_name, period_hours)
    remaining_minutes     = max(0, error_budget_minutes - consumed_minutes)
    
    %{
      budget_minutes:    error_budget_minutes,
      consumed_minutes:  consumed_minutes,
      remaining_minutes: remaining_minutes,
      remaining_percent: remaining_minutes / error_budget_minutes * 100
    }
  end

  defp fetch_metric(_query), do: 150.0  # placeholder
  defp get_downtime_minutes(_, _), do: 5.0  # placeholder
end
```

---

## Step 920: Operational Runbooks

```elixir
defmodule MyApp.Runbooks do
  # Automated runbooks for common incidents

  def high_memory do
    Logger.warning("High memory detected, running cleanup")
    
    :erlang.garbage_collect()
    
    # Force process hibernation on idle GenServers
    MyApp.ProcessRegistry.list_idle()
    |> Enum.each(fn pid ->
      :proc_lib.hibernate(pid, :hibernated, [])
    end)
    
    # Clear caches
    MyApp.ProductCache.flush()
    
    after_mem = :erlang.memory(:total) / 1_048_576
    Logger.info("Memory after cleanup: #{Float.round(after_mem, 1)} MB")
  end

  def high_cpu do
    # Check scheduler utilization
    util = :scheduler.utilization(1_000)
    Logger.warning("Scheduler utilization: #{inspect(util)}")
    
    # Find busy processes
    :recon.proc_count(:reductions, 10)
    |> Enum.each(fn {pid, reductions, info} ->
      Logger.info("Busy process #{inspect(pid)}: #{reductions} reductions - #{inspect(info)}")
    end)
  end

  def slow_queries do
    # Check pg_stat_statements
    MyApp.Repo.query!("""
      SELECT query, calls, mean_exec_time
      FROM pg_stat_statements
      ORDER BY mean_exec_time DESC
      LIMIT 10;
    """)
    |> then(& &1.rows)
    |> Enum.each(fn [query, calls, mean_ms] ->
      Logger.warning("Slow query (#{Float.round(mean_ms, 1)}ms avg, #{calls} calls): #{String.slice(query, 0, 100)}")
    end)
  end
end
```

---

## สรุป Part 85

✅ **Step 911** - Graceful shutdown  
✅ **Step 912** - Feature flags  
✅ **Step 913** - Zero-downtime deployments  
✅ **Step 914** - Error boundaries  
✅ **Step 915** - Configuration management  
✅ **Step 916** - Secrets management  
✅ **Step 917** - DB connection management  
✅ **Step 918** - Audit logging  
✅ **Step 919** - SLO monitoring  
✅ **Step 920** - Operational runbooks  

➡️ [Part 86: API Design Patterns](./part-86-api-design.md)
