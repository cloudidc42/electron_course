# Part 75: Distributed Systems Patterns (Steps 811-830)

## Step 811: Distributed Elixir Setup

```elixir
# Connect Elixir nodes across machines
# node1@host1, node2@host2

# Start with a name
iex --name myapp@192.168.1.10 --cookie my_secret_cookie

# Connect nodes
Node.connect(:"myapp@192.168.1.11")

# Check connected nodes
Node.list()  # [:"myapp@192.168.1.11"]

# production config using libcluster
# mix.exs: {:libcluster, "~> 3.3"}

config :libcluster,
  topologies: [
    k8s: [
      strategy: Cluster.Strategy.Kubernetes,
      config: [
        kubernetes_selector:   "app=my-app",
        kubernetes_node_basename: "my-app",
        polling_interval: 10_000
      ]
    ]
  ]

# application.ex
children = [
  {Cluster.Supervisor, [topologies, [name: MyApp.ClusterSupervisor]]},
  # ...
]
```

---

## Step 812: Distributed Process Registry

```elixir
# Horde: distributed process registry and supervisor
# mix.exs: {:horde, "~> 0.9"}

defmodule MyApp.HordeRegistry do
  use Horde.Registry

  def start_link(opts) do
    Horde.Registry.start_link(__MODULE__,
      [keys: :unique, members: :auto],
      name: __MODULE__
    )
  end

  def child_spec(opts) do
    %{id: __MODULE__, start: {__MODULE__, :start_link, [opts]}}
  end
end

defmodule MyApp.HordeSupervisor do
  use Horde.DynamicSupervisor

  def start_link(opts) do
    Horde.DynamicSupervisor.start_link(__MODULE__,
      [strategy: :one_for_one, members: :auto],
      name: __MODULE__
    )
  end

  def start_child(child_spec) do
    Horde.DynamicSupervisor.start_child(__MODULE__, child_spec)
  end
end

# Register a process cluster-wide
defmodule MyApp.SessionServer do
  use GenServer

  def start_link(session_id) do
    GenServer.start_link(__MODULE__, session_id,
      name: {:via, Horde.Registry, {MyApp.HordeRegistry, session_id}}
    )
  end

  def get(session_id) do
    case Horde.Registry.lookup(MyApp.HordeRegistry, session_id) do
      [{pid, _}] -> {:ok, GenServer.call(pid, :get)}
      []         -> {:error, :not_found}
    end
  end
end
```

---

## Step 813: Consistent Hashing

```elixir
defmodule MyApp.ConsistentHash do
  # Distribute work evenly across nodes/workers
  # Using a hash ring

  def new(nodes, replicas \\ 100) do
    ring = Enum.flat_map(nodes, fn node ->
      for i <- 0..replicas do
        hash = :erlang.phash2("#{node}:#{i}", 2_147_483_647)
        {hash, node}
      end
    end)
    |> Enum.sort_by(&elem(&1, 0))

    %{ring: ring, nodes: nodes}
  end

  def get_node(%{ring: ring}, key) do
    hash = :erlang.phash2(key, 2_147_483_647)

    case Enum.find(ring, fn {h, _} -> h >= hash end) do
      nil       -> ring |> List.first() |> elem(1)  # wrap around
      {_, node} -> node
    end
  end

  def add_node(%{ring: ring, nodes: nodes} = state, node, replicas \\ 100) do
    new_entries = for i <- 0..replicas do
      hash = :erlang.phash2("#{node}:#{i}", 2_147_483_647)
      {hash, node}
    end

    new_ring = (ring ++ new_entries) |> Enum.sort_by(&elem(&1, 0))
    %{state | ring: new_ring, nodes: [node | nodes]}
  end
end

# Usage
ring = MyApp.ConsistentHash.new(["worker1", "worker2", "worker3"])
MyApp.ConsistentHash.get_node(ring, "user:123")  # "worker2"
MyApp.ConsistentHash.get_node(ring, "user:456")  # "worker1"
```

---

## Step 814: Two-Phase Commit

```elixir
defmodule MyApp.TwoPhaseCommit do
  # Coordinate distributed transactions across multiple databases

  def execute(participants, transaction_fn) do
    transaction_id = generate_id()

    # Phase 1: Prepare (vote)
    case prepare_all(participants, transaction_id, transaction_fn) do
      {:ok, prepared} ->
        # Phase 2: Commit
        commit_all(prepared, transaction_id)

      {:error, reason, prepared} ->
        # Abort all that prepared
        abort_all(prepared, transaction_id)
        {:error, reason}
    end
  end

  defp prepare_all(participants, transaction_id, fun) do
    Enum.reduce_while(participants, {:ok, []}, fn {node, module}, {:ok, prepared} ->
      case :rpc.call(node, module, :prepare, [transaction_id, fun]) do
        {:ok, vote} -> {:cont, {:ok, [{node, module, vote} | prepared]}}
        {:error, _} = err -> {:halt, {:error, err, prepared}}
      end
    end)
  end

  defp commit_all(prepared, transaction_id) do
    Enum.each(prepared, fn {node, module, _vote} ->
      :rpc.cast(node, module, :commit, [transaction_id])
    end)
    :ok
  end

  defp abort_all(prepared, transaction_id) do
    Enum.each(prepared, fn {node, module, _vote} ->
      :rpc.cast(node, module, :abort, [transaction_id])
    end)
  end

  defp generate_id, do: :crypto.strong_rand_bytes(16) |> Base.encode16()
end
```

---

## Step 815: Gossip Protocol

```elixir
defmodule MyApp.Gossip do
  use GenServer

  @gossip_interval 5_000
  @fanout 3

  def start_link(_), do: GenServer.start_link(__MODULE__, %{}, name: __MODULE__)

  def broadcast(key, value) do
    GenServer.cast(__MODULE__, {:spread, key, value})
  end

  def get(key) do
    GenServer.call(__MODULE__, {:get, key})
  end

  def init(_) do
    schedule_gossip()
    {:ok, %{data: %{}, version: 0}}
  end

  def handle_cast({:spread, key, value}, state) do
    new_version = state.version + 1
    new_data    = Map.put(state.data, key, {value, new_version})
    spread_to_peers(%{key => {value, new_version}})
    {:noreply, %{state | data: new_data, version: new_version}}
  end

  def handle_call({:get, key}, _from, state) do
    case Map.get(state.data, key) do
      nil          -> {:reply, nil, state}
      {value, _}   -> {:reply, value, state}
    end
  end

  def handle_info(:gossip, state) do
    # Periodically gossip state to random peers
    spread_to_peers(state.data)
    schedule_gossip()
    {:noreply, state}
  end

  def handle_info({:gossip_data, remote_data}, state) do
    # Merge remote data, keep highest version
    merged = Map.merge(state.data, remote_data, fn _key, {v1, ver1}, {v2, ver2} ->
      if ver2 > ver1, do: {v2, ver2}, else: {v1, ver1}
    end)
    {:noreply, %{state | data: merged}}
  end

  defp spread_to_peers(data) do
    peers = Node.list() |> Enum.take_random(@fanout)
    Enum.each(peers, fn peer ->
      :rpc.cast(peer, __MODULE__, :gossip_data, [data])
    end)
  end

  defp schedule_gossip, do: Process.send_after(self(), :gossip, @gossip_interval)
end
```

---

## Step 816: Leader Election

```elixir
defmodule MyApp.LeaderElection do
  use GenServer

  # Simple leader election via sorted node names
  # For production, use raft-based algorithms

  def start_link(_), do: GenServer.start_link(__MODULE__, %{}, name: __MODULE__)

  def leader?, do: GenServer.call(__MODULE__, :leader?)
  def current_leader, do: GenServer.call(__MODULE__, :current_leader)

  def init(_) do
    :net_kernel.monitor_nodes(true)
    {:ok, %{leader: elect_leader()}}
  end

  def handle_call(:leader?, _from, %{leader: leader} = state) do
    {:reply, leader == node(), state}
  end

  def handle_call(:current_leader, _from, %{leader: leader} = state) do
    {:reply, leader, state}
  end

  def handle_info({:nodeup, _node}, state) do
    {:noreply, %{state | leader: elect_leader()}}
  end

  def handle_info({:nodedown, _node}, state) do
    {:noreply, %{state | leader: elect_leader()}}
  end

  # Simplest approach: lowest node name wins
  defp elect_leader do
    [node() | Node.list()]
    |> Enum.sort()
    |> List.first()
  end
end

# Execute only on leader
def run_if_leader(fun) do
  if MyApp.LeaderElection.leader?() do
    fun.()
  end
end
```

---

## Step 817: Distributed Counters with CRDTs

```elixir
defmodule MyApp.CRDT.GCounter do
  # Grow-only counter: merge by taking max for each node

  def new(node_id) do
    %{node_id: node_id, counters: %{node_id => 0}}
  end

  def increment(%{node_id: id, counters: counters} = counter) do
    %{counter | counters: Map.update(counters, id, 1, &(&1 + 1))}
  end

  def value(%{counters: counters}) do
    Enum.sum(Map.values(counters))
  end

  def merge(%{counters: c1} = counter, %{counters: c2}) do
    merged = Map.merge(c1, c2, fn _key, v1, v2 -> max(v1, v2) end)
    %{counter | counters: merged}
  end
end

defmodule MyApp.DistributedCounter do
  use GenServer

  def start_link(name) do
    GenServer.start_link(__MODULE__, name, name: {:global, name})
  end

  def increment(name) do
    GenServer.cast({:global, name}, :increment)
  end

  def value(name) do
    GenServer.call({:global, name}, :value)
  end

  def init(name) do
    node_id = node()
    counter = MyApp.CRDT.GCounter.new(node_id)
    
    :timer.send_interval(5_000, :sync)
    {:ok, %{name: name, counter: counter}}
  end

  def handle_cast(:increment, %{counter: counter} = state) do
    {:noreply, %{state | counter: MyApp.CRDT.GCounter.increment(counter)}}
  end

  def handle_call(:value, _from, %{counter: counter} = state) do
    {:reply, MyApp.CRDT.GCounter.value(counter), state}
  end

  def handle_info(:sync, %{name: name, counter: counter} = state) do
    Enum.each(Node.list(), fn peer ->
      :rpc.cast(peer, __MODULE__, :merge, [name, counter])
    end)
    {:noreply, state}
  end

  def handle_cast({:merge, remote_counter}, %{counter: counter} = state) do
    merged = MyApp.CRDT.GCounter.merge(counter, remote_counter)
    {:noreply, %{state | counter: merged}}
  end
end
```

---

## Step 818: Distributed Locks

```elixir
defmodule MyApp.DistributedLock do
  # Redis-based distributed lock (Redlock algorithm)

  @lock_expiry 30_000  # 30 seconds

  def acquire(resource, timeout \\ 5_000) do
    token = generate_token()
    deadline = System.monotonic_time(:millisecond) + timeout

    acquire_with_retry(resource, token, deadline)
  end

  def release(resource, token) do
    # Atomic compare-and-delete with Lua
    lua = """
    if redis.call('get', KEYS[1]) == ARGV[1] then
      return redis.call('del', KEYS[1])
    else
      return 0
    end
    """
    Redix.command(:redix, ["EVAL", lua, "1", lock_key(resource), token])
    :ok
  end

  def with_lock(resource, fun, timeout \\ 5_000) do
    case acquire(resource, timeout) do
      {:ok, token} ->
        try do
          fun.()
        after
          release(resource, token)
        end
      {:error, :timeout} ->
        {:error, :could_not_acquire_lock}
    end
  end

  defp acquire_with_retry(resource, token, deadline) do
    if System.monotonic_time(:millisecond) > deadline do
      {:error, :timeout}
    else
      case Redix.command(:redix, [
        "SET", lock_key(resource), token,
        "NX", "PX", @lock_expiry
      ]) do
        {:ok, "OK"} -> {:ok, token}
        {:ok, nil}  ->
          Process.sleep(50)
          acquire_with_retry(resource, token, deadline)
      end
    end
  end

  defp lock_key(resource), do: "lock:#{resource}"
  defp generate_token, do: :crypto.strong_rand_bytes(16) |> Base.encode16()
end
```

---

## Step 819: Conflict-Free Replicated Data

```elixir
defmodule MyApp.CRDT.LWWRegister do
  # Last-Write-Wins Register

  def new(value \\ nil) do
    %{value: value, timestamp: System.os_time(:microsecond)}
  end

  def set(register, new_value) do
    %{register | value: new_value, timestamp: System.os_time(:microsecond)}
  end

  def merge(%{timestamp: t1} = r1, %{timestamp: t2} = r2) do
    if t1 >= t2, do: r1, else: r2
  end

  def value(%{value: value}), do: value
end

defmodule MyApp.CRDT.ORSet do
  # Observed-Remove Set: add/remove with unique tags

  def new, do: %{adds: %{}, removes: MapSet.new()}

  def add(%{adds: adds} = set, element) do
    tag = :crypto.strong_rand_bytes(8) |> Base.encode16()
    %{set | adds: Map.update(adds, element, MapSet.new([tag]), &MapSet.put(&1, tag))}
  end

  def remove(%{adds: adds, removes: removes} = set, element) do
    tags = Map.get(adds, element, MapSet.new())
    %{set | removes: MapSet.union(removes, tags)}
  end

  def member?(%{adds: adds, removes: removes}, element) do
    case Map.get(adds, element) do
      nil  -> false
      tags -> not Enum.empty?(MapSet.difference(tags, removes))
    end
  end

  def merge(%{adds: a1, removes: r1}, %{adds: a2, removes: r2}) do
    merged_adds = Map.merge(a1, a2, fn _k, s1, s2 -> MapSet.union(s1, s2) end)
    %{adds: merged_adds, removes: MapSet.union(r1, r2)}
  end

  def to_list(%{adds: adds, removes: removes}) do
    adds
    |> Enum.filter(fn {_elem, tags} ->
      not Enum.empty?(MapSet.difference(tags, removes))
    end)
    |> Enum.map(&elem(&1, 0))
  end
end
```

---

## Step 820: Partition Tolerance

```elixir
defmodule MyApp.PartitionTolerant do
  # Handle network partitions with local state fallback

  def read_with_fallback(key) do
    case read_from_cluster(key) do
      {:ok, value} ->
        # Cache locally for partition scenarios
        put_local_cache(key, value)
        {:ok, value}

      {:error, :noproc} ->
        # Node is unavailable, use local cache
        case get_local_cache(key) do
          {:ok, value} ->
            Logger.warning("Using cached value for #{key} (cluster unavailable)")
            {:ok, value, :stale}
          nil ->
            {:error, :unavailable}
        end
    end
  end

  def write_with_healing(key, value) do
    case write_to_cluster(key, value) do
      :ok ->
        put_local_cache(key, value)
        :ok

      {:error, :noproc} ->
        # Write to local pending queue
        enqueue_write(key, value)
        Logger.warning("Write queued for #{key} (cluster unavailable)")
        {:ok, :queued}
    end
  end

  def replay_pending_writes do
    pending = dequeue_all_writes()
    
    Enum.each(pending, fn {key, value} ->
      case write_to_cluster(key, value) do
        :ok -> Logger.info("Replayed pending write for #{key}")
        {:error, reason} ->
          Logger.error("Failed to replay write for #{key}: #{inspect(reason)}")
          enqueue_write(key, value)  # Re-queue for next attempt
      end
    end)
  end
end
```

---

## สรุป Part 75

✅ **Step 811** - Distributed Elixir setup  
✅ **Step 812** - Distributed process registry  
✅ **Step 813** - Consistent hashing  
✅ **Step 814** - Two-phase commit  
✅ **Step 815** - Gossip protocol  
✅ **Step 816** - Leader election  
✅ **Step 817** - CRDTs (G-Counter)  
✅ **Step 818** - Distributed locks  
✅ **Step 819** - LWW Register / OR-Set  
✅ **Step 820** - Partition tolerance  

➡️ [Part 76: Stream Processing](./part-76-stream-processing.md)
