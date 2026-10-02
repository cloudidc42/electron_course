# Part 81: Microservices Architecture (Steps 871-890)

## Step 871: Service Communication

```elixir
defmodule MyApp.ServiceClient do
  # HTTP client for inter-service communication
  # mix.exs: {:req, "~> 0.5"}

  @base_headers [
    {"content-type", "application/json"},
    {"accept",       "application/json"}
  ]

  def call(service, path, method \\ :get, body \\ nil) do
    url     = service_url(service, path)
    headers = auth_headers() ++ @base_headers

    options = [
      headers:         headers,
      receive_timeout: 30_000,
      retry:           :transient,
      max_retries:     3
    ]

    options = if body, do: Keyword.put(options, :json, body), else: options

    case apply(Req, method, [url, options]) do
      {:ok, %{status: status, body: body}} when status in 200..299 ->
        {:ok, body}
      {:ok, %{status: 404}} ->
        {:error, :not_found}
      {:ok, %{status: 401}} ->
        {:error, :unauthorized}
      {:ok, %{status: status, body: body}} ->
        {:error, {:http_error, status, body}}
      {:error, exception} ->
        {:error, {:network_error, exception}}
    end
  end

  defp service_url(:user_service,    path), do: "#{System.get_env("USER_SERVICE_URL")}#{path}"
  defp service_url(:order_service,   path), do: "#{System.get_env("ORDER_SERVICE_URL")}#{path}"
  defp service_url(:payment_service, path), do: "#{System.get_env("PAYMENT_SERVICE_URL")}#{path}"

  defp auth_headers do
    token = System.get_env("SERVICE_AUTH_TOKEN", "")
    [{"x-service-token", token}, {"x-caller", "my-app"}]
  end
end
```

---

## Step 872: Service Discovery

```elixir
defmodule MyApp.ServiceRegistry do
  # Service discovery using Consul (or similar)
  
  def discover(service_name) do
    case Req.get("#{consul_url()}/v1/health/service/#{service_name}",
           params: [passing: true]) do
      {:ok, %{body: instances}} when instances != [] ->
        # Round-robin selection
        instance = Enum.random(instances)
        service  = instance["Service"]
        
        {:ok, %{
          host:   service["Address"],
          port:   service["Port"],
          id:     service["ID"],
          tags:   service["Tags"]
        }}
      {:ok, %{body: []}} ->
        {:error, :no_healthy_instances}
      {:error, _} ->
        {:error, :consul_unavailable}
    end
  end

  def register(service_name, port) do
    Req.put("#{consul_url()}/v1/agent/service/register",
      json: %{
        Name:    service_name,
        Port:    port,
        Address: local_ip(),
        Check: %{
          HTTP:     "http://#{local_ip()}:#{port}/health",
          Interval: "10s",
          Timeout:  "2s"
        }
      }
    )
  end

  def deregister(service_id) do
    Req.put("#{consul_url()}/v1/agent/service/deregister/#{service_id}")
  end

  defp consul_url, do: System.get_env("CONSUL_URL", "http://localhost:8500")
  defp local_ip,   do: "127.0.0.1"
end
```

---

## Step 873: API Gateway Pattern

```elixir
defmodule MyAppWeb.GatewayController do
  use MyAppWeb, :controller

  # API Gateway: route, authenticate, rate limit, transform

  def route(conn, %{"service" => service, "path" => path}) do
    with :ok    <- authenticate(conn),
         :ok    <- rate_limit(conn),
         target <- resolve_service(service),
         {:ok, response} <- forward_request(conn, target, path) do
      
      conn
      |> put_resp_content_type(response.headers["content-type"] || "application/json")
      |> send_resp(response.status, response.body)
    else
      {:error, :unauthorized} ->
        conn |> put_status(401) |> json(%{error: "Unauthorized"})
      {:error, :rate_limited} ->
        conn |> put_status(429) |> json(%{error: "Too many requests"})
      {:error, reason} ->
        conn |> put_status(502) |> json(%{error: "Bad gateway: #{inspect(reason)}"})
    end
  end

  defp authenticate(conn) do
    case get_req_header(conn, "authorization") do
      ["Bearer " <> token] -> MyApp.Auth.verify_token(token)
      _                    -> {:error, :unauthorized}
    end
  end

  defp rate_limit(conn) do
    ip = to_string(:inet.ntoa(conn.remote_ip))
    MyApp.RateLimit.check("gateway:#{ip}", 100, 60)
  end

  defp resolve_service("users"),    do: System.get_env("USER_SERVICE_URL")
  defp resolve_service("orders"),   do: System.get_env("ORDER_SERVICE_URL")
  defp resolve_service("payments"), do: System.get_env("PAYMENT_SERVICE_URL")
  defp resolve_service(_),          do: nil

  defp forward_request(conn, target, path) do
    method  = conn.method |> String.downcase() |> String.to_atom()
    url     = "#{target}/#{path}"
    headers = forward_headers(conn)
    
    Req.request(method: method, url: url, headers: headers, body: conn.body_params)
    |> case do
      {:ok, resp} -> {:ok, resp}
      {:error, e} -> {:error, e}
    end
  end

  defp forward_headers(conn) do
    conn.req_headers
    |> Enum.filter(fn {k, _} -> k in ~w(content-type accept x-request-id) end)
    |> Enum.concat([{"x-forwarded-for", to_string(:inet.ntoa(conn.remote_ip))}])
  end
end
```

---

## Step 874: Event-Driven Microservices

```elixir
defmodule MyApp.OrderService.EventPublisher do
  # Publish domain events to message bus (Kafka/RabbitMQ)

  def publish_order_placed(order) do
    event = %{
      type:       "order.placed",
      id:         Ecto.UUID.generate(),
      occurred_at: DateTime.utc_now(),
      data: %{
        order_id:    order.id,
        customer_id: order.customer_id,
        total:       Decimal.to_string(order.total),
        items:       format_items(order.items)
      }
    }

    case MyApp.Kafka.Producer.publish("order-events", order.id, event) do
      :ok ->
        :telemetry.execute([:events, :published], %{count: 1}, %{type: "order.placed"})
        :ok
      {:error, reason} ->
        # Outbox pattern: save to DB for retry
        MyApp.Repo.insert!(%MyApp.OutboxEvent{
          event_type: event.type,
          payload:    event,
          status:     :pending
        })
        Logger.warning("Event publish failed, saved to outbox: #{inspect(reason)}")
        :ok
    end
  end

  defp format_items(items) do
    Enum.map(items, &%{product_id: &1.product_id, quantity: &1.quantity, price: Decimal.to_string(&1.price)})
  end
end

defmodule MyApp.NotificationService.OrderConsumer do
  # React to order events from message bus

  def handle_event(%{"type" => "order.placed"} = event) do
    order_data = event["data"]
    
    MyApp.Mailer.send_order_confirmation(
      customer_id: order_data["customer_id"],
      order_id:    order_data["order_id"],
      total:       order_data["total"]
    )
  end

  def handle_event(%{"type" => "order.shipped"} = event) do
    MyApp.Notifications.push(
      event["data"]["customer_id"],
      "Your order has shipped! Tracking: #{event["data"]["tracking_number"]}"
    )
  end
end
```

---

## Step 875: Service Mesh Patterns

```elixir
defmodule MyApp.ServiceMesh do
  # Implement service mesh patterns in Elixir

  # Retry with exponential backoff
  def with_retry(fun, opts \\ []) do
    max_attempts = Keyword.get(opts, :max_attempts, 3)
    base_delay   = Keyword.get(opts, :base_delay_ms, 100)
    
    Enum.reduce_while(1..max_attempts, nil, fn attempt, _ ->
      case fun.() do
        {:ok, result} -> {:halt, {:ok, result}}
        {:error, reason} when attempt < max_attempts ->
          delay = round(base_delay * :math.pow(2, attempt - 1))
          Process.sleep(delay)
          {:cont, nil}
        {:error, reason} ->
          {:halt, {:error, reason}}
      end
    end)
  end

  # Timeout wrapper
  def with_timeout(fun, timeout_ms \\ 5_000) do
    task = Task.async(fun)
    case Task.yield(task, timeout_ms) do
      {:ok, result} -> result
      nil ->
        Task.shutdown(task, :brutal_kill)
        {:error, :timeout}
    end
  end

  # Bulkhead: limit concurrent calls to service
  def with_bulkhead(service, fun) do
    case :poolboy.checkout(:"#{service}_pool", false) do
      :full ->
        {:error, :service_overloaded}
      worker ->
        try do
          :poolboy.transaction(:"#{service}_pool", fn _worker -> fun.() end)
        after
          :poolboy.checkin(:"#{service}_pool", worker)
        end
    end
  end
end
```

---

## Step 876: Database per Service

```elixir
# Each microservice has its own Repo

# orders_service/lib/orders/repo.ex
defmodule Orders.Repo do
  use Ecto.Repo, otp_app: :orders, adapter: Ecto.Adapters.Postgres
end

# payments_service/lib/payments/repo.ex
defmodule Payments.Repo do
  use Ecto.Repo, otp_app: :payments, adapter: Ecto.Adapters.Postgres
end

# config/runtime.exs for Orders Service
config :orders, Orders.Repo,
  url:     System.fetch_env!("ORDERS_DB_URL"),
  pool_size: 10

# Cross-service data: use API calls, not DB joins
defmodule Orders.OrderWithCustomer do
  def fetch(order_id) do
    with {:ok, order} <- Orders.OrderRepo.find(order_id),
         {:ok, customer} <- Customers.ServiceClient.get(order.customer_id) do
      {:ok, Map.put(order, :customer, customer)}
    end
  end
end

# Eventual consistency: order has customer snapshot
defmodule Orders.Order do
  schema "orders" do
    field :customer_id,       :integer
    # Denormalized snapshot at time of order
    field :customer_name,     :string
    field :customer_email,    :string
    timestamps()
  end
end
```

---

## Step 877: Circuit Breaker Service

```elixir
defmodule MyApp.ServiceBreaker do
  use GenServer

  @failure_threshold 5
  @success_threshold 2
  @timeout_ms 30_000

  def start_link(service_name) do
    GenServer.start_link(__MODULE__, service_name, name: via(service_name))
  end

  def call(service_name, fun) do
    GenServer.call(via(service_name), {:call, fun}, 10_000)
  end

  def init(service_name) do
    {:ok, %{
      service:  service_name,
      state:    :closed,
      failures: 0,
      successes: 0,
      last_failure: nil
    }}
  end

  def handle_call({:call, fun}, _from, %{state: :open} = state) do
    if should_attempt_half_open?(state) do
      handle_call({:call, fun}, nil, %{state | state: :half_open})
    else
      {:reply, {:error, :circuit_open}, state}
    end
  end

  def handle_call({:call, fun}, _from, state) do
    case execute_safely(fun) do
      {:ok, result} ->
        new_state = record_success(state)
        {:reply, {:ok, result}, new_state}
      {:error, _} = error ->
        new_state = record_failure(state)
        {:reply, error, new_state}
    end
  end

  defp record_success(%{state: :half_open, successes: s} = state) when s + 1 >= @success_threshold do
    Logger.info("Circuit closed for #{state.service}")
    %{state | state: :closed, failures: 0, successes: 0}
  end
  defp record_success(%{state: :half_open} = state), do: %{state | successes: state.successes + 1}
  defp record_success(state), do: %{state | failures: 0}

  defp record_failure(%{state: :closed, failures: f} = state) when f + 1 >= @failure_threshold do
    Logger.warning("Circuit opened for #{state.service}")
    %{state | state: :open, failures: f + 1, last_failure: System.monotonic_time(:millisecond)}
  end
  defp record_failure(state) do
    %{state | state: :open, failures: state.failures + 1, last_failure: System.monotonic_time(:millisecond)}
  end

  defp should_attempt_half_open?(%{last_failure: last_failure}) do
    System.monotonic_time(:millisecond) - last_failure > @timeout_ms
  end

  defp execute_safely(fun) do
    try do
      fun.()
    rescue
      e -> {:error, e}
    end
  end

  defp via(name), do: {:via, Registry, {MyApp.BreakerRegistry, name}}
end
```

---

## Step 878: Distributed Tracing

```elixir
defmodule MyApp.Tracing do
  # OpenTelemetry for distributed tracing across services

  require OpenTelemetry.Tracer, as: Tracer

  def start_span(name, attrs \\ %{}) do
    Tracer.start_span(name, %{attributes: attrs})
  end

  def with_span(name, attrs \\ %{}, fun) do
    Tracer.with_span name, %{attributes: attrs} do
      fun.()
    end
  end

  # Propagate trace context in HTTP headers
  def inject_headers(headers) do
    :otel_propagator_text_map.inject(headers)
  end

  def extract_headers(headers) do
    :otel_propagator_text_map.extract(:otel_ctx.get_current(), headers)
  end

  # Instrument outgoing HTTP
  def traced_request(service, path, fun) do
    with_span "http.client.#{service}", %{url: path, service: service} do
      try do
        result = fun.()
        Tracer.set_attributes(%{status: "ok"})
        result
      rescue
        e ->
          Tracer.set_status(:error, inspect(e))
          reraise e, __STACKTRACE__
      end
    end
  end
end

# Usage in service client
defmodule MyApp.UserServiceClient do
  def get_user(user_id) do
    MyApp.Tracing.traced_request(:user_service, "/users/#{user_id}", fn ->
      MyApp.ServiceClient.call(:user_service, "/users/#{user_id}")
    end)
  end
end
```

---

## Step 879: Health Checks

```elixir
defmodule MyAppWeb.HealthController do
  use MyAppWeb, :controller

  def liveness(conn, _) do
    json(conn, %{status: "alive"})
  end

  def readiness(conn, _) do
    checks = %{
      database:   check_database(),
      redis:      check_redis(),
      kafka:      check_kafka(),
      disk_space: check_disk()
    }

    all_ok = Enum.all?(checks, fn {_, v} -> v == :ok end)

    status = if all_ok, do: 200, else: 503
    details = Map.new(checks, fn {k, v} -> {k, if(v == :ok, do: "up", else: "down")} end)

    conn
    |> put_status(status)
    |> json(%{status: (if all_ok, do: "healthy", else: "unhealthy"), checks: details})
  end

  defp check_database do
    case Ecto.Adapters.SQL.query(MyApp.Repo, "SELECT 1") do
      {:ok, _}    -> :ok
      {:error, _} -> :error
    end
  rescue
    _ -> :error
  end

  defp check_redis do
    case Redix.command(:redix, ["PING"]) do
      {:ok, "PONG"} -> :ok
      _             -> :error
    end
  rescue
    _ -> :error
  end

  defp check_kafka do
    :brod.get_partitions_count(:my_client, "health-check")
    |> case do
      {:ok, _}    -> :ok
      {:error, _} -> :error
    end
  rescue
    _ -> :error
  end

  defp check_disk do
    case :disksup.get_disk_data() do
      [{_, _, usage} | _] when usage < 90 -> :ok
      _ -> :error
    end
  end
end
```

---

## Step 880: Service Configuration

```elixir
defmodule MyApp.ServiceConfig do
  # Centralized config from environment + feature flags

  use GenServer

  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)

  def get(key, default \\ nil) do
    GenServer.call(__MODULE__, {:get, key, default})
  end

  def refresh do
    GenServer.cast(__MODULE__, :refresh)
  end

  def init(_) do
    :timer.send_interval(60_000, :refresh)
    {:ok, load_config()}
  end

  def handle_call({:get, key, default}, _from, config) do
    {:reply, Map.get(config, key, default), config}
  end

  def handle_cast(:refresh, _config) do
    {:noreply, load_config()}
  end

  def handle_info(:refresh, _config) do
    {:noreply, load_config()}
  end

  defp load_config do
    env_config = %{
      db_pool_size:    System.get_env("DB_POOL_SIZE", "10") |> String.to_integer(),
      cache_ttl:       System.get_env("CACHE_TTL", "3600") |> String.to_integer(),
      log_level:       System.get_env("LOG_LEVEL", "info"),
      feature_flags:   load_feature_flags()
    }
    env_config
  end

  defp load_feature_flags do
    # Load from LaunchDarkly, Unleash, or Redis
    case Redix.command(:redix, ["HGETALL", "feature_flags"]) do
      {:ok, pairs} ->
        pairs
        |> Enum.chunk_every(2)
        |> Map.new(fn [k, v] -> {k, v == "true"} end)
      _ -> %{}
    end
  end
end
```

---

## สรุป Part 81

✅ **Step 871** - Service communication  
✅ **Step 872** - Service discovery  
✅ **Step 873** - API gateway pattern  
✅ **Step 874** - Event-driven microservices  
✅ **Step 875** - Service mesh patterns  
✅ **Step 876** - Database per service  
✅ **Step 877** - Circuit breaker service  
✅ **Step 878** - Distributed tracing  
✅ **Step 879** - Health checks  
✅ **Step 880** - Service configuration  

➡️ [Part 82: Advanced Ecto Patterns](./part-82-ecto-advanced.md)
