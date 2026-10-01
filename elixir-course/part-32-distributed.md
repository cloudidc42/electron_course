# Part 32: Distributed Systems (Steps 341-360)

## Step 341: Distributed Elixir Basics

```elixir
# Start nodes:
# node1: iex --sname node1 --cookie secret -S mix
# node2: iex --sname node2 --cookie secret -S mix

# Connect nodes
Node.connect(:"node2@hostname")
Node.list()  # => [:"node2@hostname"]

# Run code on remote node
Node.spawn(:"node2@hostname", fn ->
  IO.puts("Running on #{Node.self()}")
end)

# RPC call
result = :rpc.call(:"node2@hostname", MyModule, :function, [arg1, arg2])

# Distributed process names
{:ok, pid} = GenServer.start_link(MyServer, [], name: {:global, :my_server})
GenServer.call({:global, :my_server}, :request)  # from any node
```

---

## Step 342: Horde สำหรับ Distributed Registry

```elixir
# mix.exs
{:horde, "~> 0.9"}

# Distributed Registry
defmodule MyApp.DistributedRegistry do
  use Horde.Registry
  
  def start_link(opts) do
    Horde.Registry.start_link(__MODULE__, [keys: :unique, members: :auto], opts)
  end
  
  def init(opts) do
    [members: get_members()]
    |> Keyword.merge(opts)
    |> Horde.Registry.init()
  end
  
  defp get_members do
    [Node.self() | Node.list()]
    |> Enum.map(&{__MODULE__, &1})
  end
end

# Distributed Supervisor
defmodule MyApp.DistributedSupervisor do
  use Horde.DynamicSupervisor
  
  def start_link(opts) do
    Horde.DynamicSupervisor.start_link(__MODULE__, [strategy: :one_for_one, members: :auto], opts)
  end
  
  def start_child(spec) do
    Horde.DynamicSupervisor.start_child(__MODULE__, spec)
  end
end

# Worker that lives on exactly ONE node across cluster
defmodule MyApp.GameServer do
  use GenServer
  
  def start_link(game_id) do
    name = {:via, Horde.Registry, {MyApp.DistributedRegistry, "game:#{game_id}"}}
    GenServer.start_link(__MODULE__, game_id, name: name)
  end
  
  def get_state(game_id) do
    name = {:via, Horde.Registry, {MyApp.DistributedRegistry, "game:#{game_id}"}}
    GenServer.call(name, :get_state)
  end
end
```

---

## Step 343: Phoenix PubSub Distributed

```elixir
# config/prod.exs - distributed PubSub
config :my_app, MyApp.PubSub,
  adapter: Phoenix.PubSub.PG2  # distributed via Erlang process groups

# Or via Redis (cross-language)
config :my_app, MyApp.PubSub,
  adapter: Phoenix.PubSub.Redis,
  node_name: System.get_env("PHX_NODE_NAME") || "myapp",
  redis_url: System.get_env("REDIS_URL") || "redis://localhost:6379"

# Publish from any node, all subscribers get it
Phoenix.PubSub.broadcast(MyApp.PubSub, "topic", %{event: :update})

# Broadcast from one node to all nodes
Phoenix.PubSub.broadcast_from(MyApp.PubSub, self(), "topic", %{event: :update})
```

---

## Step 344: Consistent Hashing

```elixir
defmodule MyApp.ConsistentHash do
  @ring_size 256
  
  def assign_node(key, nodes) do
    hash = :erlang.phash2(key, @ring_size)
    node_idx = rem(hash, length(nodes))
    Enum.at(nodes, node_idx)
  end
  
  def assign_primary_replica(key, nodes, replicas \\ 3) do
    hash = :erlang.phash2(key, @ring_size)
    
    Enum.map(0..(replicas - 1), fn i ->
      node_idx = rem(hash + i, length(nodes))
      Enum.at(nodes, node_idx)
    end)
    |> Enum.uniq()
  end
end

# Usage
nodes = Node.list()
responsible_node = MyApp.ConsistentHash.assign_node("user:123", nodes)

if responsible_node == Node.self() do
  process_locally()
else
  Node.spawn(responsible_node, fn -> process_locally() end)
end
```

---

## Step 345: Distributed Cache

```elixir
defmodule MyApp.DistributedCache do
  @moduledoc "Cache backed by Mnesia across nodes"
  
  def setup do
    :mnesia.create_schema([Node.self() | Node.list()])
    :mnesia.start()
    
    :mnesia.create_table(:dist_cache, [
      attributes: [:key, :value, :expires_at],
      type: :set,
      disc_copies: [Node.self() | Node.list()]  # replicated
    ])
  end
  
  def get(key) do
    :mnesia.dirty_read(:dist_cache, key)
    |> case do
      [{:dist_cache, ^key, value, exp}] when exp > now() -> {:ok, value}
      _ -> :miss
    end
  end
  
  def put(key, value, ttl \\ 300) do
    expires_at = now() + ttl
    :mnesia.dirty_write({:dist_cache, key, value, expires_at})
    :ok
  end
  
  def delete(key) do
    :mnesia.dirty_delete(:dist_cache, key)
    :ok
  end
  
  defp now, do: System.os_time(:second)
end
```

---

## Step 346: Leader Election

```elixir
defmodule MyApp.LeaderElection do
  use GenServer
  
  @election_interval 10_000
  
  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)
  
  def am_leader? do
    GenServer.call(__MODULE__, :am_leader?)
  end
  
  def init(_) do
    schedule_election()
    {:ok, %{leader: nil, term: 0}}
  end
  
  def handle_info(:elect, state) do
    new_leader = elect_leader()
    schedule_election()
    {:noreply, %{state | leader: new_leader}}
  end
  
  def handle_call(:am_leader?, _from, %{leader: leader} = state) do
    {:reply, leader == Node.self(), state}
  end
  
  defp elect_leader do
    # Simplest: sort nodes, smallest = leader
    [Node.self() | Node.list()]
    |> Enum.sort()
    |> List.first()
  end
  
  defp schedule_election do
    Process.send_after(self(), :elect, @election_interval)
  end
end

# Usage: run cron jobs only on leader
defmodule MyApp.Workers.Scheduler do
  use GenServer
  
  def handle_info(:tick, state) do
    if MyApp.LeaderElection.am_leader?() do
      do_scheduled_work()
    end
    {:noreply, state}
  end
end
```

---

## Step 347: Process Distribution ด้วย delta_crdt

```elixir
# mix.exs
{:delta_crdt, "~> 0.6"}

# Conflict-free replicated data type
defmodule MyApp.DistributedCounter do
  use GenServer
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  def init(_opts) do
    {:ok, crdt} = DeltaCrdt.start_link(DeltaCrdt.AWLWWMap, sync_interval: 200)
    
    # Connect to other nodes' CRDTs
    for node <- Node.list() do
      remote_crdt = {DeltaCrdt, node}
      DeltaCrdt.set_neighbours(crdt, [remote_crdt])
    end
    
    {:ok, %{crdt: crdt}}
  end
  
  def increment(key, amount \\ 1) do
    GenServer.call(__MODULE__, {:increment, key, amount})
  end
  
  def get(key) do
    GenServer.call(__MODULE__, {:get, key})
  end
  
  def handle_call({:increment, key, amount}, _from, %{crdt: crdt} = state) do
    current = DeltaCrdt.get(crdt, key) || 0
    DeltaCrdt.put(crdt, key, current + amount)
    {:reply, :ok, state}
  end
  
  def handle_call({:get, key}, _from, %{crdt: crdt} = state) do
    {:reply, DeltaCrdt.get(crdt, key) || 0, state}
  end
end
```

---

## Step 348: Network Partitions

```elixir
defmodule MyApp.PartitionHandler do
  @moduledoc """
  Handle network partitions (split-brain).
  CAP Theorem: choose Consistency OR Availability.
  
  Strategy options:
  - CP (choose consistency): stop writes when quorum lost
  - AP (choose availability): continue with stale data, merge later
  """
  
  def handle_partition_heal(nodes) do
    require Logger
    Logger.warning("Partition healed, syncing with nodes: #{inspect(nodes)}")
    
    # Trigger data sync
    for node <- nodes do
      Node.spawn(node, MyApp.DataSync, :sync_from, [Node.self()])
    end
  end
  
  # Quorum check
  def has_quorum?(total_nodes) do
    active_nodes = length([Node.self() | Node.list()])
    active_nodes > div(total_nodes, 2)
  end
end

# Application callback
defmodule MyApp.Application do
  def start(_type, _args) do
    :net_kernel.monitor_nodes(true)
    # ...
  end
end

# In GenServer - detect node join/leave
def handle_info({:nodeup, node}, state) do
  require Logger
  Logger.info("Node joined cluster: #{node}")
  MyApp.PartitionHandler.handle_partition_heal([node])
  {:noreply, state}
end

def handle_info({:nodedown, node}, state) do
  require Logger
  Logger.warning("Node left cluster: #{node}")
  {:noreply, state}
end
```

---

## Step 349: Distributed Tracing

```elixir
# mix.exs
{:opentelemetry, "~> 1.3"},
{:opentelemetry_exporter, "~> 1.6"},
{:opentelemetry_phoenix, "~> 1.2"},
{:opentelemetry_ecto, "~> 1.2"}

# config/prod.exs
config :opentelemetry_exporter,
  otlp_protocol: :http_protobuf,
  otlp_endpoint: "http://otel-collector:4318"

config :opentelemetry,
  span_processor: :batch,
  resource: [service: [name: "my-elixir-app"]]

# application.ex
def start(_type, _args) do
  OpentelemetryPhoenix.setup()
  OpentelemetryEcto.setup([:my_app, :repo])
  :opentelemetry.register_tracer(:my_app, "1.0.0")
  # ...
end

# Custom spans
defmodule MyApp.OrderService do
  require OpenTelemetry.Tracer, as: Tracer
  
  def process_order(order_id) do
    Tracer.with_span "order.process", %{attributes: [{"order.id", order_id}]} do
      with {:ok, order} <- fetch_order(order_id),
           {:ok, _}     <- Tracer.with_span("payment.charge") do
                             charge_payment(order)
                           end do
        Tracer.set_attribute("order.status", "completed")
        {:ok, order}
      end
    end
  end
end
```

---

## Step 350: Saga Pattern

```elixir
defmodule MyApp.OrderSaga do
  @moduledoc """
  Distributed transaction using Saga pattern.
  Each step has a compensating action.
  """
  
  defstruct [:order_id, :steps, :completed, :state]
  
  def run(order_params) do
    saga = %__MODULE__{
      order_id: UUID.uuid4(),
      steps: steps(),
      completed: [],
      state: %{params: order_params}
    }
    
    execute(saga)
  end
  
  defp steps do
    [
      {:reserve_inventory, &reserve_inventory/1, &release_inventory/1},
      {:charge_payment,    &charge_payment/1,    &refund_payment/1},
      {:create_order,      &create_order/1,      &cancel_order/1},
      {:send_confirmation, &send_confirmation/1, &noop/1}
    ]
  end
  
  defp execute(%{steps: []} = saga), do: {:ok, saga.state}
  defp execute(%{steps: [{name, action, _} | rest]} = saga) do
    case action.(saga.state) do
      {:ok, new_state} ->
        execute(%{saga |
          steps: rest,
          completed: [{name, saga.state} | saga.completed],
          state: new_state
        })
      
      {:error, reason} ->
        compensate(saga, reason)
    end
  end
  
  defp compensate(%{completed: []} = _saga, reason) do
    {:error, reason}
  end
  defp compensate(%{completed: [{_, prev_state, compensate_fn} | rest]} = saga, reason) do
    compensate_fn.(prev_state)
    compensate(%{saga | completed: rest}, reason)
  end
  
  defp reserve_inventory(state) do
    case Inventory.reserve(state.params.items) do
      {:ok, reservation} -> {:ok, Map.put(state, :reservation_id, reservation.id)}
      error -> error
    end
  end
  
  defp release_inventory(state) do
    Inventory.release(state.reservation_id)
  end
  
  defp noop(state), do: {:ok, state}
end
```

---

## สรุป Part 32

✅ **Step 341** - Distributed Elixir basics  
✅ **Step 342** - Horde registry/supervisor  
✅ **Step 343** - PubSub distributed  
✅ **Step 344** - Consistent hashing  
✅ **Step 345** - Distributed cache (Mnesia)  
✅ **Step 346** - Leader election  
✅ **Step 347** - CRDT  
✅ **Step 348** - Network partitions  
✅ **Step 349** - Distributed tracing  
✅ **Step 350** - Saga pattern  

➡️ [Part 33: Event Sourcing และ CQRS](./part-33-event-sourcing.md)
