# Part 48: Interview & Career (Steps 531-560)

## Step 531: Common Interview Questions

```elixir
# Q: What is the difference between a process and a thread?
# A:
# Elixir Process:
# - Lightweight (~2KB stack, grows dynamically)
# - Message passing only (no shared memory)
# - Supervised and fault-isolated
# - Erlang VM managed
# - Millions can run concurrently
#
# OS Thread:
# - Heavy (~1MB stack)
# - Shared memory (requires locks/mutexes)
# - Crash can affect whole program
# - OS managed
# - Limited by CPU cores

# Q: Explain pattern matching
def process({:ok, value}),   do: handle_success(value)
def process({:error, reason}), do: handle_error(reason)
def process(%User{role: :admin} = user), do: admin_action(user)
def process(%User{} = user),  do: user_action(user)
```

---

## Step 532: BEAM/OTP Interview Questions

```elixir
# Q: What is OTP?
# A: Open Telecom Platform - set of libraries + design principles for
#    building reliable distributed systems. Key components:
#    - GenServer: generic server behavior
#    - Supervisor: fault tolerance
#    - Application: lifecycle management
#    - Registry: process naming

# Q: When would you use Agent vs GenServer?
# Agent: when you need simple state storage with no complex logic
# GenServer: when you need complex logic, multiple callbacks, or handle_info

# Agent example:
defmodule Counter do
  use Agent
  def start_link(initial), do: Agent.start_link(fn -> initial end, name: __MODULE__)
  def increment, do: Agent.update(__MODULE__, &(&1 + 1))
  def value, do: Agent.get(__MODULE__, & &1)
end

# Q: What is a supervision tree?
# A: Hierarchical tree of processes where supervisors restart children
#    on failure. Enables "let it crash" philosophy.

# Q: Explain :one_for_one vs :one_for_all supervisor strategies
# :one_for_one - only restart the crashed child
# :one_for_all - restart all children when one crashes
# :rest_for_one - restart crashed child + all started after it
```

---

## Step 533: Functional Programming Questions

```elixir
# Q: What is immutability and why does it matter?
# A: Data cannot be modified after creation.
#    - Thread safe (no shared mutable state)
#    - Easier to reason about
#    - Enables pattern matching

data = %{name: "Alice"}
new_data = Map.put(data, :age, 30)
# data is still %{name: "Alice"}, not modified

# Q: Explain the pipe operator
"  hello world  "
|> String.trim()
|> String.split(" ")
|> Enum.map(&String.capitalize/1)
|> Enum.join(" ")
# "Hello World"

# Q: What is a closure?
multiplier = fn x ->
  fn n -> n * x end
end
double = multiplier.(2)
double.(5) # 10

# Q: Recursion vs Enum
# Tail-recursive (efficient)
defmodule MyList do
  def sum(list, acc \\ 0)
  def sum([], acc), do: acc
  def sum([h | t], acc), do: sum(t, acc + h)
end

# Usually prefer Enum
Enum.sum([1, 2, 3, 4, 5])
```

---

## Step 534: Phoenix/Ecto Questions

```elixir
# Q: Explain Phoenix request lifecycle
# Router → Endpoint (plugs) → Controller → View → Template
# OR: Router → LiveView

# Q: What is a changeset?
user_params = %{name: "", email: "invalid"}

changeset = %User{}
  |> Ecto.Changeset.cast(user_params, [:name, :email])
  |> Ecto.Changeset.validate_required([:name, :email])
  |> Ecto.Changeset.validate_format(:email, ~r/@/)

changeset.valid?  # false
changeset.errors  # [name: {"can't be blank", ...}, email: {"has invalid format", ...}]

# Q: How do you prevent N+1 queries?
# Bad: N+1
posts = Repo.all(Post)
Enum.map(posts, fn post -> post.comments end)  # N queries!

# Good: preload
posts = Post |> Repo.all() |> Repo.preload(:comments)  # 2 queries

# Or in query
posts = from(p in Post, preload: [:comments]) |> Repo.all()

# Q: What is Ecto.Multi?
# Atomic multi-step database operations with rollback on failure
```

---

## Step 535: Concurrency Questions

```elixir
# Q: How do you handle concurrent updates?
# Option 1: Serialized through GenServer
defmodule Counter do
  use GenServer
  def increment, do: GenServer.call(__MODULE__, :increment)
  def handle_call(:increment, _from, n), do: {:reply, n + 1, n + 1}
end

# Option 2: Database optimistic locking
{count, _} = Repo.update_all(
  from(p in Product, where: p.id == ^id and p.stock > 0),
  [inc: [stock: -1]],
  returning: true
)
if count == 0, do: {:error, :out_of_stock}

# Option 3: ETS atomic operations
:ets.update_counter(:table, :key, {2, -1, 0, 0})  # min=0

# Q: What is back-pressure?
# Mechanism to slow producers when consumers can't keep up
# GenStage: demand-driven, consumers request work
# Broadway: built-in back-pressure for data pipelines

# Q: Task vs GenServer?
# Task: short-lived async work, use Task.async/await
# GenServer: long-lived state management
result = Task.async(fn -> heavy_computation() end) |> Task.await()
```

---

## Step 536: System Design Interview

```
Design a real-time messaging system (like Slack) with Elixir:

1. ARCHITECTURE
   ┌─────────────┐   WebSocket   ┌──────────────┐
   │   Client    │◄─────────────►│ Phoenix App  │
   └─────────────┘               └──────┬───────┘
                                         │
                              ┌──────────▼──────────┐
                              │  Phoenix Channels    │
                              │  + Presence         │
                              └──────────┬──────────┘
                                         │
                   ┌─────────────────────┼──────────────────┐
                   ▼                     ▼                  ▼
            ┌──────────┐        ┌──────────────┐   ┌────────────┐
            │ Messages │        │ Presence     │   │   Files    │
            │  Service │        │  Service     │   │  Service   │
            └──────────┘        └──────────────┘   └────────────┘

2. ELIXIR STRENGTHS
   - Phoenix Channels: WebSocket multiplexing
   - Presence: distributed online tracking (CRDT-based)
   - PubSub: fan-out message delivery
   - BEAM: millions of lightweight processes

3. SCALABILITY
   - Multiple nodes via libcluster
   - Redis PubSub adapter for cross-node
   - Partitioned channels by workspace ID
   - Message fanout via process tree

4. STORAGE
   - PostgreSQL for message history (append-only)
   - Redis for typing indicators, online status
   - S3 for file attachments
   - Elasticsearch for search
```

---

## Step 537: Code Review Best Practices

```elixir
# 1. Pattern matching over conditionals
# Bad
def process(result) do
  if result == :ok do
    # ...
  else
    # ...
  end
end

# Good
def process(:ok),       do: handle_success()
def process({:error, r}), do: handle_error(r)

# 2. Avoid deep nesting (use with)
# Bad
def create_order(params) do
  case validate(params) do
    {:ok, valid} ->
      case check_inventory(valid) do
        {:ok, _} ->
          case charge(params.user_id) do
            {:ok, charge} ->
              {:ok, save_order(valid, charge)}
            {:error, r} -> {:error, r}
          end
        {:error, r} -> {:error, r}
      end
    {:error, r} -> {:error, r}
  end
end

# Good
def create_order(params) do
  with {:ok, valid}   <- validate(params),
       {:ok, _}       <- check_inventory(valid),
       {:ok, charge}  <- charge(params.user_id),
       {:ok, order}   <- save_order(valid, charge) do
    {:ok, order}
  end
end

# 3. Named functions vs anonymous
# Bad
Enum.map(users, fn user -> User.full_name(user) end)

# Good
Enum.map(users, &User.full_name/1)
```

---

## Step 538: Career Path in Elixir

```
Elixir Career Levels:

Junior (0-2 years):
- Elixir basics + OTP fundamentals
- Phoenix CRUD apps
- Basic Ecto queries
- ExUnit testing

Mid-level (2-5 years):
- Advanced OTP (gen_statem, dynamic supervisors)
- Phoenix LiveView + Channels
- Performance optimization
- Docker deployment
- CI/CD

Senior (5+ years):
- Distributed systems design
- Event sourcing / CQRS
- ML with Nx/Axon
- System architecture
- Mentoring

World-class / Expert:
- Open source contributions
- Conference talks (ElixirConf, Code BEAM)
- Library creation
- Core team/community leadership
- Novel architecture patterns

Resources:
- Programming Phoenix (Chris McCord)
- Elixir in Action (Sasa Juric)
- Metaprogramming Elixir (Chris McCord)
- The Little Elixir & OTP Guidebook
- ElixirForum.com
- HexDocs.pm
- ElixirConf talks on YouTube
```

---

## Step 539: Open Source Contributions

```bash
# Contributing to Elixir ecosystem
# 1. Find a project on GitHub
#    - elixir-lang/elixir (language itself)
#    - phoenixframework/phoenix
#    - elixir-ecto/ecto
#    - dashbitco/broadway
#    - exhausted ones in hex.pm

# 2. Start small: fix docs, typos, tests
# 3. Report issues with MRE (minimal reproducible example)
# 4. Implement a feature from issues
# 5. Create your own library

# Good first projects to create:
# - Mix task plugin
# - Phoenix plug library
# - Ecto extension (custom types, fragments)
# - Telemetry event handler
# - Broadway producer

# Example: create a plug
defmodule Plug.RequestId do
  @behaviour Plug
  
  def init(opts), do: opts
  
  def call(conn, _opts) do
    request_id = Plug.Conn.get_req_header(conn, "x-request-id")
      |> List.first()
      |> then(&(&1 || generate_id()))
    
    conn
    |> Plug.Conn.put_resp_header("x-request-id", request_id)
    |> Plug.Conn.assign(:request_id, request_id)
  end
  
  defp generate_id, do: :crypto.strong_rand_bytes(16) |> Base.encode16(case: :lower)
end
```

---

## Step 540: Final Project Checklist

```bash
# Complete production-ready Elixir application checklist

# Code Quality
mix format             # consistent formatting
mix credo --strict     # code analysis
mix dialyzer           # type checking
mix test --cover       # test coverage

# Security
mix deps.audit         # check vulnerable deps
mix sobelow            # security analysis

# Documentation
mix docs               # generate ExDoc
# open doc/index.html

# Build
mix release            # create OTP release
docker build . -t app  # Docker image

# Deploy
fly deploy             # or
kubectl apply -f k8s/  # Kubernetes

# Monitor
# Grafana dashboard: http://grafana:3000
# Jaeger traces: http://jaeger:16686
# Prometheus: http://prometheus:9090
```

---

## สรุป Part 48

✅ **Step 531** - Process vs thread  
✅ **Step 532** - BEAM/OTP interviews  
✅ **Step 533** - Functional programming  
✅ **Step 534** - Phoenix/Ecto  
✅ **Step 535** - Concurrency  
✅ **Step 536** - System design  
✅ **Step 537** - Code review  
✅ **Step 538** - Career path  
✅ **Step 539** - Open source  
✅ **Step 540** - Final checklist  

➡️ [Part 49: Ecosystem Deep Dive](./part-49-ecosystem.md)
