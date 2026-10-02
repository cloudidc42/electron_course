# Part 65: gRPC & Protocol Buffers (Steps 711-730)

## Step 711: gRPC Overview

```
gRPC vs REST:

REST                          gRPC
─────────────────────         ──────────────────────────
JSON (text)                   Protocol Buffers (binary)
HTTP/1.1 (usually)            HTTP/2 (multiplexed)
Request-Response              Unary + Streaming RPCs
Loose contract                Strict .proto schema
No generated code             Code generated from .proto
~5x larger payload            ~5x smaller payload
Browser native                Requires proxy for web

Use cases for gRPC:
✅ Microservice-to-service communication
✅ High-throughput internal APIs
✅ Real-time streaming
✅ Mobile backends
✅ Polyglot systems (auto-generates clients in 12+ languages)
```

---

## Step 712: Protocol Buffers

```protobuf
// priv/proto/user.proto
syntax = "proto3";

package myapp.users;
option elixir_module_prefix = "MyApp.Proto";

service UserService {
  rpc GetUser      (GetUserRequest)    returns (User);
  rpc CreateUser   (CreateUserRequest) returns (User);
  rpc ListUsers    (ListUsersRequest)  returns (stream User);
  rpc UpdateUsers  (stream UpdateUserRequest) returns (UpdateSummary);
  rpc WatchUser    (WatchUserRequest)  returns (stream UserEvent);
}

message User {
  uint64 id          = 1;
  string name        = 2;
  string email       = 3;
  string role        = 4;
  int64  created_at  = 5;
}

message GetUserRequest {
  uint64 id = 1;
}

message CreateUserRequest {
  string name     = 1;
  string email    = 2;
  string password = 3;
}

message ListUsersRequest {
  uint32 page       = 1;
  uint32 per_page   = 2;
  string filter     = 3;
}

message UpdateUserRequest {
  uint64 id   = 1;
  string name = 2;
}

message UpdateSummary {
  uint32 updated = 1;
  uint32 failed  = 2;
}

message WatchUserRequest {
  uint64 id = 1;
}

message UserEvent {
  enum Type {
    UPDATED   = 0;
    DELETED   = 1;
    LOGGED_IN = 2;
  }
  Type   type = 1;
  User   user = 2;
  int64  at   = 3;
}
```

---

## Step 713: Generate Elixir Code from Proto

```bash
# Install protoc and grpc plugins
brew install protobuf
mix archive.install hex protobuf

# Generate Elixir code
protoc \
  --elixir_out=lib/my_app/proto \
  --grpc_elixir_out=lib/my_app/proto \
  --proto_path=priv/proto \
  priv/proto/user.proto
```

```elixir
# mix.exs
{:grpc, "~> 0.7"},
{:protobuf, "~> 0.11"},
{:cowlib, "~> 2.12", override: true}

# Generated: lib/my_app/proto/user.pb.ex (message structs)
# Generated: lib/my_app/proto/user_service.pb.ex (service stubs)
```

---

## Step 714: gRPC Server

```elixir
defmodule MyApp.GRPC.UserServer do
  use GRPC.Server, service: MyApp.Proto.UserService.Service

  # Unary RPC
  def get_user(request, _stream) do
    case MyApp.Accounts.get_user(request.id) do
      nil  ->
        raise GRPC.RPCError, status: GRPC.Status.not_found(), message: "User not found"
      user ->
        user_to_proto(user)
    end
  end

  # Unary RPC
  def create_user(request, _stream) do
    case MyApp.Accounts.create_user(%{
      name:     request.name,
      email:    request.email,
      password: request.password
    }) do
      {:ok, user}       -> user_to_proto(user)
      {:error, changeset} ->
        msg = MyApp.ChangesetErrors.translate(changeset)
        raise GRPC.RPCError, status: GRPC.Status.invalid_argument(), message: msg
    end
  end

  # Server streaming RPC
  def list_users(request, stream) do
    MyApp.Accounts.stream_users(
      page:     request.page,
      per_page: request.per_page,
      filter:   request.filter
    )
    |> Stream.each(fn user ->
      GRPC.Server.send_reply(stream, user_to_proto(user))
    end)
    |> Stream.run()
  end

  # Client streaming RPC
  def update_users(request_stream, _stream) do
    {updated, failed} =
      request_stream
      |> Enum.reduce({0, 0}, fn request, {u, f} ->
        case MyApp.Accounts.update_user(request.id, %{name: request.name}) do
          {:ok, _}    -> {u + 1, f}
          {:error, _} -> {u, f + 1}
        end
      end)

    MyApp.Proto.UpdateSummary.new(updated: updated, failed: failed)
  end

  # Bidirectional streaming
  def watch_user(request_stream, stream) do
    user_id = request_stream |> Enum.take(1) |> List.first() |> Map.get(:id)
    
    Phoenix.PubSub.subscribe(MyApp.PubSub, "user:#{user_id}")
    
    receive do
      {:user_updated, user} ->
        event = MyApp.Proto.UserEvent.new(
          type: :UPDATED,
          user: user_to_proto(user),
          at:   System.os_time(:second)
        )
        GRPC.Server.send_reply(stream, event)
    end
  end

  defp user_to_proto(user) do
    MyApp.Proto.User.new(
      id:         user.id,
      name:       user.name,
      email:      user.email,
      role:       to_string(user.role),
      created_at: DateTime.to_unix(user.inserted_at)
    )
  end
end

# Start gRPC endpoint in application.ex
defmodule MyApp.Application do
  def start(_type, _args) do
    children = [
      {GRPC.Server.Supervisor, {MyApp.GRPC.Endpoint, 50051}},
      # ... other children
    ]
    Supervisor.start_link(children, strategy: :one_for_one)
  end
end

defmodule MyApp.GRPC.Endpoint do
  use GRPC.Endpoint
  intercept GRPC.Logger.Server
  run MyApp.GRPC.UserServer
end
```

---

## Step 715: gRPC Client

```elixir
defmodule MyApp.GRPC.Client do
  alias MyApp.Proto.{UserService, GetUserRequest, CreateUserRequest, ListUsersRequest}

  def channel(host \\ "localhost", port \\ 50051) do
    {:ok, channel} = GRPC.Stub.connect("#{host}:#{port}")
    channel
  end

  def get_user(id, channel \\ channel()) do
    request = GetUserRequest.new(id: id)
    case UserService.Stub.get_user(channel, request) do
      {:ok, user}             -> {:ok, proto_to_map(user)}
      {:error, %GRPC.RPCError{status: 5}} -> {:error, :not_found}
      {:error, reason}        -> {:error, reason}
    end
  end

  def create_user(attrs, channel \\ channel()) do
    request = CreateUserRequest.new(
      name:     attrs.name,
      email:    attrs.email,
      password: attrs.password
    )
    case UserService.Stub.create_user(channel, request) do
      {:ok, user}          -> {:ok, proto_to_map(user)}
      {:error, error}      -> {:error, error.message}
    end
  end

  # Server streaming: returns a Stream
  def stream_users(opts \\ [], channel \\ channel()) do
    request = ListUsersRequest.new(
      page:     Keyword.get(opts, :page, 1),
      per_page: Keyword.get(opts, :per_page, 50)
    )

    {:ok, stream} = UserService.Stub.list_users(channel, request)
    
    stream
    |> Enum.map(&proto_to_map/1)
  end

  defp proto_to_map(proto) do
    %{
      id:    proto.id,
      name:  proto.name,
      email: proto.email,
      role:  proto.role
    }
  end
end
```

---

## Step 716: Interceptors / Middleware

```elixir
defmodule MyApp.GRPC.AuthInterceptor do
  @behaviour GRPC.ServerInterceptor

  def call(request, stream, next, _opts) do
    token = GRPC.Stream.get_headers(stream)["authorization"]

    case authenticate(token) do
      {:ok, user} ->
        stream = GRPC.Stream.put_private(stream, :current_user, user)
        next.(request, stream)

      {:error, _} ->
        raise GRPC.RPCError,
          status: GRPC.Status.unauthenticated(),
          message: "Invalid or missing token"
    end
  end

  defp authenticate("Bearer " <> token) do
    MyApp.Auth.verify_token(token)
  end
  defp authenticate(_), do: {:error, :no_token}
end

defmodule MyApp.GRPC.LoggingInterceptor do
  @behaviour GRPC.ServerInterceptor

  def call(request, stream, next, _opts) do
    start   = System.monotonic_time(:millisecond)
    method  = stream.method_name
    user_id = GRPC.Stream.get_private(stream, :current_user)

    result  = next.(request, stream)

    elapsed = System.monotonic_time(:millisecond) - start
    Logger.info("gRPC #{method} user=#{user_id} #{elapsed}ms")

    result
  end
end

# Register interceptors
defmodule MyApp.GRPC.Endpoint do
  use GRPC.Endpoint
  intercept MyApp.GRPC.LoggingInterceptor
  intercept MyApp.GRPC.AuthInterceptor
  run MyApp.GRPC.UserServer
end
```

---

## Step 717: gRPC-Web (Browser Support)

```elixir
# gRPC-Web proxy in Elixir
# Browsers can't use HTTP/2 directly for gRPC
# Use a proxy or grpc-web protocol

# In your Phoenix router/endpoint, add gRPC-Web transcoder
defmodule MyAppWeb.GRPCWebPlug do
  import Plug.Conn

  def call(%Plug.Conn{path_info: ["grpc" | _]} = conn, _opts) do
    # Transcode gRPC-web to gRPC
    content_type = get_req_header(conn, "content-type") |> List.first()
    
    if String.starts_with?(content_type || "", "application/grpc-web") do
      proxy_to_grpc(conn)
    else
      conn
    end
  end

  def call(conn, _opts), do: conn

  defp proxy_to_grpc(conn) do
    # Forward to gRPC server on port 50051
    # This is a simplified example; use a proper proxy in production
    conn
  end
end
```

---

## Step 718: Proto Validation

```elixir
defmodule MyApp.GRPC.Validation do
  def validate_create_user(%{name: name, email: email, password: password}) do
    errors = []
    errors = if String.length(name) < 2, do: ["name too short" | errors], else: errors
    errors = if not valid_email?(email), do: ["invalid email" | errors], else: errors
    errors = if String.length(password) < 8, do: ["password too short" | errors], else: errors

    case errors do
      [] -> :ok
      _  -> {:error, Enum.join(errors, ", ")}
    end
  end

  defp valid_email?(email), do: email =~ ~r/^[^\s@]+@[^\s@]+\.[^\s@]+$/
end

# Usage in server
def create_user(request, _stream) do
  case MyApp.GRPC.Validation.validate_create_user(request) do
    :ok ->
      # proceed
    {:error, msg} ->
      raise GRPC.RPCError, status: GRPC.Status.invalid_argument(), message: msg
  end
end
```

---

## Step 719: Health Check Protocol

```elixir
defmodule MyApp.GRPC.HealthServer do
  # Implements standard gRPC health check protocol
  # grpc.health.v1.Health
  
  use GRPC.Server, service: Grpc.Health.V1.Health.Service

  def check(%{service: ""}, _stream) do
    # Overall health
    %Grpc.Health.V1.HealthCheckResponse{status: :SERVING}
  end

  def check(%{service: service}, _stream) do
    status = case check_service(service) do
      :ok    -> :SERVING
      :error -> :NOT_SERVING
    end

    %Grpc.Health.V1.HealthCheckResponse{status: status}
  end

  defp check_service("UserService") do
    case MyApp.Repo.query("SELECT 1") do
      {:ok, _} -> :ok
      _        -> :error
    end
  end

  defp check_service(_), do: :ok
end
```

---

## Step 720: Testing gRPC Services

```elixir
defmodule MyApp.GRPC.UserServerTest do
  use ExUnit.Case

  setup do
    {:ok, channel} = GRPC.Stub.connect("localhost:50051")
    {:ok, channel: channel}
  end

  test "get_user returns user", %{channel: channel} do
    user = insert(:user)
    request = MyApp.Proto.GetUserRequest.new(id: user.id)
    
    {:ok, response} = MyApp.Proto.UserService.Stub.get_user(channel, request)
    
    assert response.id   == user.id
    assert response.name == user.name
  end

  test "get_user raises NOT_FOUND for missing user", %{channel: channel} do
    request = MyApp.Proto.GetUserRequest.new(id: 99999)
    
    {:error, error} = MyApp.Proto.UserService.Stub.get_user(channel, request)
    
    assert error.status == GRPC.Status.not_found()
  end

  test "list_users streams all users", %{channel: channel} do
    users = insert_list(3, :user)
    request = MyApp.Proto.ListUsersRequest.new(per_page: 10)
    
    {:ok, stream} = MyApp.Proto.UserService.Stub.list_users(channel, request)
    results = Enum.to_list(stream)
    
    assert length(results) == 3
    assert Enum.map(results, & &1.id) |> Enum.sort() ==
           Enum.map(users, & &1.id) |> Enum.sort()
  end
end
```

---

## สรุป Part 65

✅ **Step 711** - gRPC overview  
✅ **Step 712** - Protocol Buffers  
✅ **Step 713** - Code generation  
✅ **Step 714** - gRPC server  
✅ **Step 715** - gRPC client  
✅ **Step 716** - Interceptors / middleware  
✅ **Step 717** - gRPC-Web  
✅ **Step 718** - Validation  
✅ **Step 719** - Health check protocol  
✅ **Step 720** - Testing gRPC  

➡️ [Part 66: Rate Limiting & Throttling](./part-66-rate-limiting.md)
