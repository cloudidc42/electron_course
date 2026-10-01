# Part 46: Advanced OTP Patterns (Steps 511-530)

## Step 511: Process Pools

```elixir
defmodule MyApp.ProcessPool do
  use Supervisor
  
  @pool_size 10
  
  def start_link(opts) do
    Supervisor.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  def checkout do
    GenServer.call(MyApp.PoolManager, :checkout)
  end
  
  def checkin(worker) do
    GenServer.cast(MyApp.PoolManager, {:checkin, worker})
  end
  
  def transaction(fun) do
    worker = checkout()
    
    try do
      fun.(worker)
    after
      checkin(worker)
    end
  end
  
  def init(_opts) do
    workers = for i <- 1..@pool_size do
      id = :"worker_#{i}"
      Supervisor.child_spec({MyApp.Worker, name: id}, id: id)
    end
    
    children = workers ++ [{MyApp.PoolManager, workers: @pool_size}]
    Supervisor.init(children, strategy: :one_for_one)
  end
end

defmodule MyApp.PoolManager do
  use GenServer
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  def init(opts) do
    size    = Keyword.get(opts, :workers, 10)
    workers = for i <- 1..size, do: :"worker_#{i}"
    
    {:ok, %{available: :queue.from_list(workers), waiting: :queue.new()}}
  end
  
  def handle_call(:checkout, from, %{available: avail, waiting: wait} = state) do
    case :queue.out(avail) do
      {{:value, worker}, rest} ->
        {:reply, worker, %{state | available: rest}}
      
      {:empty, _} ->
        {:noreply, %{state | waiting: :queue.in(from, wait)}}
    end
  end
  
  def handle_cast({:checkin, worker}, %{available: avail, waiting: wait} = state) do
    case :queue.out(wait) do
      {{:value, caller}, rest} ->
        GenServer.reply(caller, worker)
        {:noreply, %{state | waiting: rest}}
      
      {:empty, _} ->
        {:noreply, %{state | available: :queue.in(worker, avail)}}
    end
  end
end
```

---

## Step 512: Finite State Machine (gen_statem)

```elixir
defmodule MyApp.Connection do
  @behaviour :gen_statem
  
  # States: :disconnected, :connecting, :connected, :reconnecting
  
  def start_link(opts) do
    :gen_statem.start_link({:local, __MODULE__}, __MODULE__, opts, [])
  end
  
  def connect(pid) do
    :gen_statem.call(pid, :connect)
  end
  
  def disconnect(pid) do
    :gen_statem.call(pid, :disconnect)
  end
  
  def send_data(pid, data) do
    :gen_statem.call(pid, {:send, data})
  end
  
  @impl :gen_statem
  def callback_mode, do: [:state_functions, :state_enter]
  
  @impl :gen_statem
  def init(opts) do
    {:ok, :disconnected, %{host: opts[:host], port: opts[:port], socket: nil}}
  end
  
  # State: disconnected
  def disconnected(:enter, _old_state, data) do
    {:keep_state, data}
  end
  
  def disconnected({:call, from}, :connect, data) do
    {:next_state, :connecting, data, [{:reply, from, :ok}]}
  end
  
  # State: connecting
  def connecting(:enter, _old_state, %{host: host, port: port} = data) do
    {:ok, socket} = :gen_tcp.connect(host, port, [], 5_000)
    {:next_state, :connected, %{data | socket: socket}}
  rescue
    _ -> {:next_state, :disconnected, data, [{:state_timeout, 5_000, :retry}]}
  end
  
  # State: connected
  def connected(:enter, _old_state, data) do
    {:keep_state, data}
  end
  
  def connected({:call, from}, {:send, data_to_send}, %{socket: socket} = state) do
    case :gen_tcp.send(socket, data_to_send) do
      :ok    -> {:keep_state, state, [{:reply, from, :ok}]}
      {:error, reason} ->
        {:next_state, :reconnecting, %{state | socket: nil},
          [{:reply, from, {:error, reason}}]}
    end
  end
  
  def connected({:call, from}, :disconnect, %{socket: socket} = state) do
    :gen_tcp.close(socket)
    {:next_state, :disconnected, %{state | socket: nil}, [{:reply, from, :ok}]}
  end
  
  def connected(:info, {:tcp_closed, _socket}, state) do
    {:next_state, :reconnecting, %{state | socket: nil}}
  end
  
  # State: reconnecting
  def reconnecting(:enter, _old_state, data) do
    {:keep_state, data, [{:state_timeout, 2_000, :reconnect}]}
  end
  
  def reconnecting(:state_timeout, :reconnect, data) do
    {:next_state, :connecting, data}
  end
end
```

---

## Step 513: Saga Pattern

```elixir
defmodule MyApp.Saga do
  defmodule Step do
    defstruct [:name, :execute, :compensate]
  end
  
  def run(steps) do
    Enum.reduce_while(steps, [], fn step, completed ->
      case step.execute.() do
        {:ok, result} ->
          {:cont, [{step, result} | completed]}
        
        {:error, reason} ->
          # Compensate in reverse order
          Enum.each(completed, fn {completed_step, result} ->
            if completed_step.compensate do
              completed_step.compensate.(result)
            end
          end)
          
          {:halt, {:error, step.name, reason}}
      end
    end)
    |> case do
      [] -> {:ok, []}
      completed when is_list(completed) -> {:ok, Enum.map(completed, fn {_step, r} -> r end)}
      error -> error
    end
  end
end

# Usage in Order creation
defmodule Orders.CreateOrder do
  alias MyApp.Saga
  
  def execute(user_id, items) do
    steps = [
      %Saga.Step{
        name: :create_order,
        execute: fn ->
          Orders.Repo.insert(Order.changeset(%Order{}, %{user_id: user_id, items: items}))
        end,
        compensate: fn order ->
          Orders.Repo.delete(order)
        end
      },
      %Saga.Step{
        name: :reserve_inventory,
        execute: fn ->
          Catalog.reserve_inventory(items)
        end,
        compensate: fn reservation ->
          Catalog.release_reservation(reservation)
        end
      },
      %Saga.Step{
        name: :charge_payment,
        execute: fn ->
          Payments.charge(user_id, calculate_total(items))
        end,
        compensate: fn charge ->
          Payments.refund(charge.id)
        end
      }
    ]
    
    Saga.run(steps)
  end
end
```

---

## Step 514: Circuit Breaker (Advanced)

```elixir
defmodule MyApp.CircuitBreaker do
  use GenServer
  
  @states [:closed, :open, :half_open]
  
  defstruct [
    :name,
    state:           :closed,
    failure_count:   0,
    success_count:   0,
    last_failure_at: nil,
    config:          %{}
  ]
  
  defmodule Config do
    defstruct [
      failure_threshold:  5,
      success_threshold:  2,
      reset_timeout:      30_000,
      call_timeout:       5_000
    ]
  end
  
  def start_link(name, config \\ %Config{}) do
    GenServer.start_link(__MODULE__, {name, config}, name: via(name))
  end
  
  def call(name, fun) do
    GenServer.call(via(name), {:call, fun}, :infinity)
  end
  
  def state(name) do
    GenServer.call(via(name), :state)
  end
  
  def init({name, config}) do
    {:ok, %__MODULE__{name: name, config: config}}
  end
  
  def handle_call(:state, _from, state) do
    {:reply, state.state, state}
  end
  
  def handle_call({:call, _fun}, _from, %{state: :open} = state) do
    if should_attempt_reset?(state) do
      {:reply, {:error, :circuit_open}, %{state | state: :half_open}}
    else
      {:reply, {:error, :circuit_open}, state}
    end
  end
  
  def handle_call({:call, fun}, _from, state) do
    task = Task.async(fn ->
      try do
        {:ok, fun.()}
      rescue
        e -> {:error, e}
      end
    end)
    
    result = Task.await(task, state.config.call_timeout)
    
    new_state = case result do
      {:ok, _}    -> record_success(state)
      {:error, _} -> record_failure(state)
    end
    
    {:reply, result, new_state}
  catch
    :exit, {:timeout, _} ->
      {:reply, {:error, :timeout}, record_failure(state)}
  end
  
  defp record_success(%{state: :half_open, success_count: n, config: %{success_threshold: t}} = s)
    when n + 1 >= t do
    %{s | state: :closed, failure_count: 0, success_count: 0}
  end
  defp record_success(%{success_count: n} = s), do: %{s | success_count: n + 1}
  
  defp record_failure(%{failure_count: n, config: %{failure_threshold: t}} = s)
    when n + 1 >= t do
    %{s | state: :open, failure_count: n + 1, last_failure_at: System.monotonic_time(:millisecond)}
  end
  defp record_failure(%{failure_count: n} = s) do
    %{s | failure_count: n + 1, last_failure_at: System.monotonic_time(:millisecond)}
  end
  
  defp should_attempt_reset?(%{last_failure_at: t, config: %{reset_timeout: timeout}}) do
    System.monotonic_time(:millisecond) - t > timeout
  end
  
  defp via(name), do: {:via, Registry, {MyApp.Registry, {:circuit_breaker, name}}}
end
```

---

## Step 515: Bulkhead Pattern

```elixir
defmodule MyApp.Bulkhead do
  @moduledoc "Isolate failures by limiting concurrent calls"
  
  use GenServer
  
  def start_link(name, max_concurrent) do
    GenServer.start_link(__MODULE__, max_concurrent, name: via(name))
  end
  
  def call(name, fun, timeout \\ 5_000) do
    case GenServer.call(via(name), :acquire, timeout) do
      :ok ->
        try do
          fun.()
        after
          GenServer.cast(via(name), :release)
        end
      
      {:error, :at_capacity} ->
        {:error, :bulkhead_full}
    end
  end
  
  def init(max_concurrent) do
    {:ok, %{max: max_concurrent, current: 0, waiting: :queue.new()}}
  end
  
  def handle_call(:acquire, from, %{current: n, max: max} = state) when n >= max do
    # Queue waiting callers
    {:noreply, %{state | waiting: :queue.in(from, state.waiting)}}
  end
  
  def handle_call(:acquire, _from, %{current: n} = state) do
    {:reply, :ok, %{state | current: n + 1}}
  end
  
  def handle_cast(:release, %{current: n, waiting: wait} = state) do
    case :queue.out(wait) do
      {{:value, from}, rest} ->
        GenServer.reply(from, :ok)
        {:noreply, %{state | waiting: rest}}
      {:empty, _} ->
        {:noreply, %{state | current: max(n - 1, 0)}}
    end
  end
  
  defp via(name), do: {:via, Registry, {MyApp.Registry, {:bulkhead, name}}}
end

# Usage with multiple isolation groups
defmodule MyApp.ExternalServices do
  def call_payment(fun) do
    MyApp.Bulkhead.call(:payment_service, fun)
  end
  
  def call_email(fun) do
    MyApp.Bulkhead.call(:email_service, fun)
  end
end
```

---

## Step 516: Back-Pressure with GenStage

```elixir
defmodule MyApp.EventIngester do
  use GenStage
  
  # Producer
  def start_link(_), do: GenStage.start_link(__MODULE__, :ok, name: __MODULE__)
  
  def ingest(event) do
    GenStage.call(__MODULE__, {:push, event})
  end
  
  def init(:ok) do
    {:producer, %{queue: :queue.new(), demand: 0}}
  end
  
  def handle_call({:push, event}, _from, %{queue: q, demand: d} = state) do
    new_queue = :queue.in(event, q)
    {events, new_state} = dispatch_events(%{state | queue: new_queue})
    {:reply, :ok, events, new_state}
  end
  
  def handle_demand(demand, state) do
    {events, new_state} = dispatch_events(%{state | demand: state.demand + demand})
    {:noreply, events, new_state}
  end
  
  defp dispatch_events(%{queue: q, demand: d} = state) when d > 0 do
    case :queue.out(q) do
      {{:value, event}, rest} ->
        {more, new_state} = dispatch_events(%{state | queue: rest, demand: d - 1})
        {[event | more], new_state}
      {:empty, _} ->
        {[], state}
    end
  end
  defp dispatch_events(state), do: {[], state}
end

defmodule MyApp.EventProcessor do
  use GenStage
  
  def start_link(_), do: GenStage.start_link(__MODULE__, :ok, name: __MODULE__)
  
  def init(:ok) do
    {:consumer, :ok,
      subscribe_to: [{MyApp.EventIngester, max_demand: 100, min_demand: 50}]}
  end
  
  def handle_events(events, _from, state) do
    Enum.each(events, &process_event/1)
    {:noreply, [], state}
  end
  
  defp process_event(event) do
    # Handle event
    :telemetry.execute([:my_app, :event, :processed], %{}, %{type: event.type})
  end
end
```

---

## Step 517: Supervisor Strategies

```elixir
defmodule MyApp.CriticalSupervisor do
  use Supervisor
  
  def start_link(_), do: Supervisor.start_link(__MODULE__, [], name: __MODULE__)
  
  def init(_) do
    children = [
      # :one_for_one: restart only crashed child
      MyApp.DatabaseWorker,
      MyApp.CacheWorker,
      
      # Rest for One: restart crashed + all started after it
      # use when later processes depend on earlier
    ]
    
    Supervisor.init(children, strategy: :one_for_one, max_restarts: 3, max_seconds: 5)
  end
end

defmodule MyApp.DependentSupervisor do
  use Supervisor
  
  def init(_) do
    children = [
      MyApp.DatabaseConn,  # if this crashes, restart it + everything after
      MyApp.QueryCache,    # depends on DatabaseConn
      MyApp.APIServer      # depends on both
    ]
    
    # :rest_for_one: crash DatabaseConn → restart QueryCache + APIServer too
    Supervisor.init(children, strategy: :rest_for_one)
  end
end

defmodule MyApp.AllForOneSupervisor do
  use Supervisor
  
  def init(_) do
    children = [
      MyApp.LeaderElection,
      MyApp.Consensus,
      MyApp.ReplicatedState
    ]
    
    # :one_for_all: any crash restarts ALL
    # Use for tightly coupled processes
    Supervisor.init(children, strategy: :one_for_all)
  end
end
```

---

## Step 518: ETS Optimization

```elixir
defmodule MyApp.Cache do
  @table_name :app_cache
  
  def setup do
    :ets.new(@table_name, [
      :set,
      :public,
      :named_table,
      read_concurrency:  true,   # readers don't block each other
      write_concurrency: true,   # writers don't block (at cost of some memory)
      decentralized_counters: true  # OTP 24+: faster counters
    ])
  end
  
  def get(key) do
    case :ets.lookup(@table_name, key) do
      [{^key, value, expiry}] ->
        if expiry > System.os_time(:second), do: {:ok, value}, else: :miss
      [] -> :miss
    end
  end
  
  def put(key, value, ttl_seconds \\ 300) do
    :ets.insert(@table_name, {key, value, System.os_time(:second) + ttl_seconds})
    :ok
  end
  
  def delete(key) do
    :ets.delete(@table_name, key)
  end
  
  # Scan and delete expired entries
  def cleanup do
    now = System.os_time(:second)
    :ets.select_delete(@table_name, [{{:_, :_, :"$1"}, [{:<, :"$1", now}], [true]}])
  end
  
  # Atomic compare-and-swap
  def update_counter(key, delta \\ 1) do
    :ets.update_counter(@table_name, key, {2, delta}, {key, 0, :infinity})
  end
end
```

---

## Step 519: Process Hibernation

```elixir
defmodule MyApp.LongLivedServer do
  use GenServer
  
  @idle_threshold 60_000  # 1 minute
  
  def start_link(opts), do: GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  
  def init(opts) do
    {:ok, %{last_active: System.monotonic_time(:millisecond), data: opts}}
  end
  
  def handle_call(:get_data, _from, state) do
    {:reply, state.data, %{state | last_active: System.monotonic_time(:millisecond)}}
  end
  
  # After handling message, check if should hibernate
  def handle_info(:check_idle, state) do
    elapsed = System.monotonic_time(:millisecond) - state.last_active
    
    if elapsed >= @idle_threshold do
      # Hibernate: GC'd, minimal memory footprint until next message
      {:noreply, state, :hibernate}
    else
      schedule_idle_check()
      {:noreply, state}
    end
  end
  
  # Wake up from hibernate on any message
  def handle_info(:wake_from_hibernate, state) do
    schedule_idle_check()
    {:noreply, %{state | last_active: System.monotonic_time(:millisecond)}}
  end
  
  defp schedule_idle_check do
    Process.send_after(self(), :check_idle, @idle_threshold)
  end
end
```

---

## Step 520: Node Monitoring

```elixir
defmodule MyApp.NodeMonitor do
  use GenServer
  require Logger
  
  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)
  
  def init(_) do
    :net_kernel.monitor_nodes(true, node_type: :all)
    nodes = Node.list()
    Logger.info("NodeMonitor started. Connected nodes: #{inspect(nodes)}")
    {:ok, %{nodes: MapSet.new(nodes)}}
  end
  
  def handle_info({:nodeup, node, _info}, state) do
    Logger.info("Node joined: #{node}")
    
    :telemetry.execute([:cluster, :node, :up], %{}, %{node: node})
    
    # Rebalance if needed
    MyApp.ClusterRebalancer.node_joined(node)
    
    {:noreply, %{state | nodes: MapSet.put(state.nodes, node)}}
  end
  
  def handle_info({:nodedown, node, _info}, state) do
    Logger.warning("Node left: #{node}")
    
    :telemetry.execute([:cluster, :node, :down], %{}, %{node: node})
    
    # Redistribute processes that were on the down node
    MyApp.ClusterRebalancer.node_left(node)
    
    {:noreply, %{state | nodes: MapSet.delete(state.nodes, node)}}
  end
  
  def handle_call(:cluster_size, _from, state) do
    {:reply, MapSet.size(state.nodes) + 1, state}  # +1 for self
  end
end
```

---

## สรุป Part 46

✅ **Step 511** - Process pools  
✅ **Step 512** - gen_statem FSM  
✅ **Step 513** - Saga pattern  
✅ **Step 514** - Circuit breaker advanced  
✅ **Step 515** - Bulkhead pattern  
✅ **Step 516** - GenStage back-pressure  
✅ **Step 517** - Supervisor strategies  
✅ **Step 518** - ETS optimization  
✅ **Step 519** - Process hibernation  
✅ **Step 520** - Node monitoring  

➡️ [Part 47: World-Class Patterns](./part-47-worldclass.md)
