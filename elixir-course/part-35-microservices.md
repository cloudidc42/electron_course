# Part 35: Microservices (Steps 381-400)

## Step 381: Microservices Architecture

```
Microservices vs Monolith:
- Monolith: simpler start, harder to scale specific parts
- Microservices: independent scaling, deploy, team ownership
- Phoenix Umbrella: middle ground (single repo, separate apps)

Communication patterns:
- Synchronous: HTTP REST, gRPC
- Asynchronous: Kafka, RabbitMQ, AMQP
- Service mesh: Envoy/Istio for service discovery, mTLS

Elixir strengths for microservices:
- Lightweight: small Docker images
- Fault tolerant: crash in one service ≠ crash all
- Hot code reload: possible zero-downtime
- PubSub native: easy event broadcast
```

---

## Step 382: Umbrella Project

```
mix new my_platform --umbrella

my_platform/
├── apps/
│   ├── accounts/       ← User service
│   ├── orders/         ← Order service
│   ├── inventory/      ← Inventory service
│   ├── notifications/  ← Notification service
│   └── api_gateway/    ← Phoenix API gateway
├── config/
└── mix.exs
```

```elixir
# apps/accounts/mix.exs
defmodule Accounts.MixProject do
  use Mix.Project
  
  def project do
    [
      app: :accounts,
      version: "0.1.0",
      build_path: "../../_build",
      config_path: "../../config/config.exs",
      deps_path: "../../deps",
      lockfile: "../../mix.lock",
      elixir: "~> 1.16",
      deps: deps()
    ]
  end
  
  defp deps do
    [
      {:phoenix, "~> 1.7"},
      {:ecto_sql, "~> 3.11"},
      {:postgrex, ">= 0.0.0"}
    ]
  end
end

# apps/api_gateway/mix.exs - depends on other apps
defp deps do
  [
    {:accounts, in_umbrella: true},
    {:orders, in_umbrella: true},
    {:phoenix, "~> 1.7"}
  ]
end
```

---

## Step 383: Service Communication (HTTP)

```elixir
# HTTP client for service-to-service
defmodule MyApp.ServiceClient do
  @timeout 5_000
  
  def get(service, path, opts \\ []) do
    url = base_url(service) <> path
    headers = [{"content-type", "application/json"} | auth_headers()]
    
    case Finch.build(:get, url, headers) |> Finch.request(MyApp.Finch, receive_timeout: @timeout) do
      {:ok, %{status: status, body: body}} when status in 200..299 ->
        {:ok, Jason.decode!(body)}
      
      {:ok, %{status: 404}} ->
        {:error, :not_found}
      
      {:ok, %{status: status, body: body}} ->
        {:error, %{status: status, body: Jason.decode!(body)}}
      
      {:error, %Mint.TransportError{reason: reason}} ->
        {:error, {:transport_error, reason}}
    end
  end
  
  def post(service, path, body, opts \\ []) do
    url = base_url(service) <> path
    headers = [{"content-type", "application/json"} | auth_headers()]
    
    Finch.build(:post, url, headers, Jason.encode!(body))
    |> Finch.request(MyApp.Finch, receive_timeout: @timeout)
    |> handle_response()
  end
  
  defp base_url(:accounts),     do: System.get_env("ACCOUNTS_SERVICE_URL", "http://accounts:4001")
  defp base_url(:orders),       do: System.get_env("ORDERS_SERVICE_URL", "http://orders:4002")
  defp base_url(:inventory),    do: System.get_env("INVENTORY_SERVICE_URL", "http://inventory:4003")
  defp base_url(:notifications), do: System.get_env("NOTIFICATIONS_URL", "http://notifications:4004")
  
  defp auth_headers do
    token = MyApp.ServiceAuth.get_token()
    [{"authorization", "Bearer #{token}"}]
  end
  
  defp handle_response({:ok, %{status: s, body: b}}) when s in 200..299, do: {:ok, Jason.decode!(b)}
  defp handle_response({:ok, %{status: 404}}), do: {:error, :not_found}
  defp handle_response({:ok, %{status: s, body: b}}), do: {:error, %{status: s, body: b}}
  defp handle_response({:error, reason}), do: {:error, reason}
end
```

---

## Step 384: gRPC ด้วย GRPC

```elixir
# mix.exs
{:grpc, "~> 0.8"},
{:protobuf, "~> 0.12"}

# priv/protos/user.proto
# syntax = "proto3";
# package myapp;
# 
# service UserService {
#   rpc GetUser (GetUserRequest) returns (UserResponse);
#   rpc CreateUser (CreateUserRequest) returns (UserResponse);
# }
# message GetUserRequest { int64 id = 1; }
# message UserResponse {
#   int64 id = 1; string email = 2; string name = 3;
# }

# Generate: mix grpc.gen --from priv/protos/user.proto

defmodule MyApp.UserServiceServer do
  use GRPC.Server, service: Myapp.UserService.Service
  
  def get_user(%{id: id}, _stream) do
    case MyApp.Accounts.get_user(id) do
      nil  -> raise GRPC.RPCError, status: GRPC.Status.not_found(), message: "User not found"
      user -> to_response(user)
    end
  end
  
  def create_user(request, _stream) do
    case MyApp.Accounts.create_user(%{email: request.email, name: request.name}) do
      {:ok, user}    -> to_response(user)
      {:error, _cs}  -> raise GRPC.RPCError, status: GRPC.Status.invalid_argument()
    end
  end
  
  defp to_response(user) do
    %Myapp.UserResponse{id: user.id, email: user.email, name: user.name}
  end
end

# gRPC client
defmodule MyApp.UserServiceClient do
  def get_user(id) do
    {:ok, channel} = GRPC.Stub.connect("accounts-service:50051")
    Myapp.UserService.Stub.get_user(channel, %Myapp.GetUserRequest{id: id})
  end
end
```

---

## Step 385: API Gateway

```elixir
defmodule ApiGatewayWeb.Router do
  use ApiGatewayWeb, :router
  
  pipeline :api do
    plug :accepts, ["json"]
    plug ApiGatewayWeb.Plugs.RateLimit
    plug ApiGatewayWeb.Plugs.Authenticate
    plug ApiGatewayWeb.Plugs.TraceRequest
  end
  
  scope "/api/v1" do
    pipe_through :api
    
    # Route to accounts service
    scope "/users" do
      get  "/",    Proxy.AccountsController, :list
      post "/",    Proxy.AccountsController, :create
      get  "/:id", Proxy.AccountsController, :show
    end
    
    # Route to orders service
    scope "/orders" do
      get  "/",    Proxy.OrdersController, :list
      post "/",    Proxy.OrdersController, :create
      get  "/:id", Proxy.OrdersController, :show
    end
    
    # Aggregated response (BFF - Backend for Frontend)
    get "/dashboard", DashboardController, :index
  end
end

defmodule ApiGatewayWeb.DashboardController do
  use ApiGatewayWeb, :controller
  
  def index(conn, _params) do
    user = conn.assigns.current_user
    
    # Parallel calls to multiple services
    [orders_task, inventory_task, notifications_task] = [
      Task.async(fn -> MyApp.ServiceClient.get(:orders, "/users/#{user.id}/orders") end),
      Task.async(fn -> MyApp.ServiceClient.get(:inventory, "/alerts") end),
      Task.async(fn -> MyApp.ServiceClient.get(:notifications, "/unread?user_id=#{user.id}") end)
    ]
    
    {:ok, orders}        = Task.await(orders_task, 5_000)
    {:ok, inventory}     = Task.await(inventory_task, 5_000)
    {:ok, notifications} = Task.await(notifications_task, 5_000)
    
    json(conn, %{
      orders:        orders,
      inventory:     inventory,
      notifications: notifications
    })
  end
end
```

---

## Step 386: Service Discovery

```elixir
defmodule MyApp.ServiceRegistry do
  use GenServer
  
  @consul_url "http://consul:8500"
  
  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)
  
  def get_service(name) do
    GenServer.call(__MODULE__, {:get_service, name})
  end
  
  def init(_) do
    schedule_refresh()
    {:ok, %{services: %{}}}
  end
  
  def handle_info(:refresh, state) do
    services = fetch_from_consul()
    schedule_refresh()
    {:noreply, %{state | services: services}}
  end
  
  def handle_call({:get_service, name}, _from, %{services: services} = state) do
    case Map.get(services, name) do
      nil      -> {:reply, {:error, :not_found}, state}
      instances ->
        instance = Enum.random(instances)  # simple round-robin
        {:reply, {:ok, instance}, state}
    end
  end
  
  defp fetch_from_consul do
    case Finch.build(:get, "#{@consul_url}/v1/catalog/services")
         |> Finch.request(MyApp.Finch) do
      {:ok, %{body: body}} ->
        Jason.decode!(body)
        |> Map.keys()
        |> Enum.reduce(%{}, fn service, acc ->
          instances = fetch_service_instances(service)
          Map.put(acc, service, instances)
        end)
      _ -> %{}
    end
  end
  
  defp fetch_service_instances(service) do
    {:ok, %{body: body}} = Finch.build(:get, "#{@consul_url}/v1/catalog/service/#{service}")
    |> Finch.request(MyApp.Finch)
    
    Jason.decode!(body)
    |> Enum.map(fn s -> "#{s["ServiceAddress"]}:#{s["ServicePort"]}" end)
  end
  
  defp schedule_refresh do
    Process.send_after(self(), :refresh, 30_000)
  end
end
```

---

## Step 387: Service-to-Service Auth

```elixir
defmodule MyApp.ServiceAuth do
  @token_ttl 3600  # 1 hour
  
  def get_token do
    case :ets.lookup(:service_tokens, :current) do
      [{:current, token, exp}] when exp > System.os_time(:second) ->
        token
      _ ->
        token = generate_token()
        :ets.insert(:service_tokens, {:current, token, System.os_time(:second) + @token_ttl - 60})
        token
    end
  end
  
  defp generate_token do
    claims = %{
      "sub"  => System.get_env("SERVICE_NAME", "unknown"),
      "iat"  => System.os_time(:second),
      "exp"  => System.os_time(:second) + @token_ttl,
      "type" => "service"
    }
    
    Joken.generate_and_sign!(claims, service_signer())
  end
  
  defp service_signer do
    secret = System.fetch_env!("SERVICE_SECRET_KEY")
    Joken.Signer.create("HS256", secret)
  end
  
  def verify_token(token) do
    secret = System.fetch_env!("SERVICE_SECRET_KEY")
    signer = Joken.Signer.create("HS256", secret)
    
    case Joken.verify_and_validate(token, signer) do
      {:ok, %{"type" => "service"} = claims} -> {:ok, claims}
      _ -> {:error, :invalid_token}
    end
  end
end
```

---

## Step 388: Health Checks ใน Microservices

```elixir
defmodule MyApp.HealthCheck do
  def check do
    dependencies = %{
      database:     check_db(),
      redis:        check_redis(),
      kafka:        check_kafka(),
      accounts_svc: check_service(:accounts, "/health"),
      orders_svc:   check_service(:orders, "/health")
    }
    
    all_healthy = Enum.all?(Map.values(dependencies), &(&1.status == :ok))
    
    %{
      status:       if(all_healthy, do: :healthy, else: :degraded),
      version:      Application.spec(:my_app, :vsn) |> to_string(),
      node:         Node.self(),
      dependencies: dependencies
    }
  end
  
  defp check_db do
    case MyApp.Repo.query("SELECT 1") do
      {:ok, _}    -> %{status: :ok}
      {:error, e} -> %{status: :error, message: Exception.message(e)}
    end
  rescue
    _ -> %{status: :error, message: "DB unreachable"}
  end
  
  defp check_service(name, path) do
    case MyApp.ServiceClient.get(name, path) do
      {:ok, %{"status" => "healthy"}} -> %{status: :ok}
      {:ok, body}                     -> %{status: :degraded, details: body}
      {:error, reason}                -> %{status: :error, message: inspect(reason)}
    end
  end
  
  defp check_redis do
    case Redix.command(:redis, ["PING"]) do
      {:ok, "PONG"} -> %{status: :ok}
      _             -> %{status: :error}
    end
  end
  
  defp check_kafka do
    # Check Kafka connectivity
    %{status: :ok}
  end
end
```

---

## Step 389: Distributed Config

```elixir
defmodule MyApp.Config do
  @moduledoc "Runtime config from Consul KV or Env"
  
  def get(key, default \\ nil) do
    case Application.get_env(:my_app, key) do
      nil   -> fetch_from_consul(key) || default
      value -> value
    end
  end
  
  def get!(key) do
    case get(key) do
      nil   -> raise "Required config #{key} not found"
      value -> value
    end
  end
  
  defp fetch_from_consul(key) do
    case Finch.build(:get, "http://consul:8500/v1/kv/my_app/#{key}")
         |> Finch.request(MyApp.Finch) do
      {:ok, %{status: 200, body: body}} ->
        body
        |> Jason.decode!()
        |> List.first()
        |> Map.get("Value")
        |> Base.decode64!()
      _ -> nil
    end
  end
end
```

---

## Step 390: Monitoring Microservices

```elixir
defmodule MyApp.ServiceMesh do
  @moduledoc "Sidecar-pattern metrics and tracing"
  
  defmodule MetricSidecar do
    def record_request(service, method, path, status, duration_ms) do
      :telemetry.execute(
        [:service_mesh, :request],
        %{duration: duration_ms},
        %{
          source_service: System.get_env("SERVICE_NAME"),
          target_service: to_string(service),
          method:         method,
          path:           path,
          status:         to_string(status)
        }
      )
    end
    
    def record_error(service, reason) do
      :telemetry.execute(
        [:service_mesh, :error],
        %{count: 1},
        %{service: to_string(service), reason: inspect(reason)}
      )
    end
  end
end

# Wrap service client with metrics
defmodule MyApp.InstrumentedClient do
  alias MyApp.ServiceMesh.MetricSidecar
  
  def get(service, path, opts \\ []) do
    start = System.monotonic_time(:millisecond)
    result = MyApp.ServiceClient.get(service, path, opts)
    duration = System.monotonic_time(:millisecond) - start
    
    status = case result do
      {:ok, _}         -> 200
      {:error, :not_found} -> 404
      {:error, _}      -> 500
    end
    
    MetricSidecar.record_request(service, "GET", path, status, duration)
    result
  end
end
```

---

## สรุป Part 35

✅ **Step 381** - Microservices concepts  
✅ **Step 382** - Umbrella projects  
✅ **Step 383** - HTTP service communication  
✅ **Step 384** - gRPC with Protobuf  
✅ **Step 385** - API Gateway + BFF  
✅ **Step 386** - Service discovery (Consul)  
✅ **Step 387** - Service-to-service auth  
✅ **Step 388** - Microservice health checks  
✅ **Step 389** - Distributed config  
✅ **Step 390** - Monitoring  

➡️ [Part 36: Machine Learning ด้วย Nx และ Axon](./part-36-ml-nx.md)
