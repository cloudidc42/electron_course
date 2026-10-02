# Part 79: Advanced OTP Patterns (Steps 851-870)

## Step 851: GenServer Advanced Patterns

```elixir
defmodule MyApp.StateMachine do
  use GenServer

  # FSM using GenServer with explicit state transitions
  @transitions %{
    idle:       [:running],
    running:    [:paused, :completed, :failed],
    paused:     [:running, :cancelled],
    completed:  [],
    failed:     [],
    cancelled:  []
  }

  def start_link(job), do: GenServer.start_link(__MODULE__, job)

  def transition(pid, to) do
    GenServer.call(pid, {:transition, to})
  end

  def state(pid), do: GenServer.call(pid, :state)

  def init(job) do
    {:ok, %{state: :idle, job: job, history: []}}
  end

  def handle_call({:transition, to}, _from, %{state: current} = state) do
    allowed = Map.get(@transitions, current, [])

    if to in allowed do
      new_state = %{state |
        state:   to,
        history: [{current, to, DateTime.utc_now()} | state.history]
      }
      {:reply, {:ok, to}, new_state}
    else
      {:reply, {:error, {:invalid_transition, current, to}}, state}
    end
  end

  def handle_call(:state, _from, state) do
    {:reply, state.state, state}
  end
end
```

---

## Step 852: Process Hibernation

```elixir
defmodule MyApp.LongLivedWorker do
  use GenServer

  @idle_timeout 30_000

  def init(state) do
    schedule_work()
    {:ok, state}
  end

  def handle_info(:work, state) do
    result = do_expensive_work(state)
    schedule_work()
    # Hibernate after work to reclaim memory during idle periods
    {:noreply, update(state, result), {:hibernate}}
  end

  # Called when a message arrives after hibernation
  # Process is woken up automatically
  def handle_call(:get_result, _from, state) do
    {:reply, state.result, state}
  end

  defp schedule_work do
    Process.send_after(self(), :work, @idle_timeout)
  end
end

# Manually hibernate a process
# The process wakes up when the next message arrives
:proc_lib.hibernate(MyApp.LongLivedWorker, :handle_info, [:work, state])
```

---

## Step 853: Dynamic Supervisor

```elixir
defmodule MyApp.WorkerPool do
  use DynamicSupervisor

  def start_link(_), do: DynamicSupervisor.start_link(__MODULE__, [], name: __MODULE__)

  def init(_), do: DynamicSupervisor.init(strategy: :one_for_one)

  def start_worker(args) do
    spec = {MyApp.Worker, args}
    DynamicSupervisor.start_child(__MODULE__, spec)
  end

  def stop_worker(pid) do
    DynamicSupervisor.terminate_child(__MODULE__, pid)
  end

  def list_workers do
    DynamicSupervisor.which_children(__MODULE__)
  end

  def worker_count do
    DynamicSupervisor.count_children(__MODULE__).workers
  end
end

defmodule MyApp.Worker do
  use GenServer

  def start_link(args) do
    GenServer.start_link(__MODULE__, args)
  end

  def init(%{task: task, callback: callback}) do
    # Process task asynchronously
    Process.send_after(self(), :process, 0)
    {:ok, %{task: task, callback: callback}}
  end

  def handle_info(:process, %{task: task, callback: callback}) do
    result = execute(task)
    callback.(result)
    {:stop, :normal, nil}
  end
end
```

---

## Step 854: Process Monitoring Patterns

```elixir
defmodule MyApp.ProcessWatcher do
  use GenServer

  def start_link(_), do: GenServer.start_link(__MODULE__, %{}, name: __MODULE__)

  def watch(pid, name) do
    GenServer.call(__MODULE__, {:watch, pid, name})
  end

  def unwatch(pid) do
    GenServer.call(__MODULE__, {:unwatch, pid})
  end

  def init(_) do
    {:ok, %{monitors: %{}}}
  end

  def handle_call({:watch, pid, name}, _from, %{monitors: monitors} = state) do
    ref = Process.monitor(pid)
    {:reply, :ok, %{state | monitors: Map.put(monitors, ref, %{pid: pid, name: name})}}
  end

  def handle_call({:unwatch, pid}, _from, %{monitors: monitors} = state) do
    case Enum.find(monitors, fn {_ref, info} -> info.pid == pid end) do
      {ref, _} ->
        Process.demonitor(ref, [:flush])
        {:reply, :ok, %{state | monitors: Map.delete(monitors, ref)}}
      nil ->
        {:reply, {:error, :not_watched}, state}
    end
  end

  def handle_info({:DOWN, ref, :process, pid, reason}, %{monitors: monitors} = state) do
    case Map.pop(monitors, ref) do
      {%{name: name}, new_monitors} ->
        Logger.warning("Process #{name} (#{inspect(pid)}) died: #{inspect(reason)}")
        notify_death(name, pid, reason)
        {:noreply, %{state | monitors: new_monitors}}
      {nil, _} ->
        {:noreply, state}
    end
  end

  defp notify_death(name, pid, reason) do
    Phoenix.PubSub.broadcast(MyApp.PubSub, "processes", {:process_died, name, pid, reason})
  end
end
```

---

## Step 855: ETS Advanced Patterns

```elixir
defmodule MyApp.ETSCache do
  @table :my_cache

  def setup do
    :ets.new(@table, [
      :named_table,
      :public,
      :set,
      read_concurrency:  true,
      write_concurrency: true
    ])
  end

  def get(key) do
    case :ets.lookup(@table, key) do
      [{^key, value, expires}] ->
        if System.os_time(:second) < expires do
          {:ok, value}
        else
          :ets.delete(@table, key)
          {:error, :expired}
        end
      [] -> {:error, :not_found}
    end
  end

  def put(key, value, ttl \\ 3600) do
    expires = System.os_time(:second) + ttl
    :ets.insert(@table, {key, value, expires})
    :ok
  end

  def delete(key) do
    :ets.delete(@table, key)
    :ok
  end

  def get_or_set(key, fun, ttl \\ 3600) do
    case get(key) do
      {:ok, value} -> value
      _ ->
        value = fun.()
        put(key, value, ttl)
        value
    end
  end

  # Match spec for finding expired entries
  def cleanup_expired do
    now = System.os_time(:second)
    :ets.select_delete(@table, [
      {{:_, :_, :"$1"}, [{:<, :"$1", now}], [true]}
    ])
  end

  # Counter operations
  def increment(key, amount \\ 1) do
    :ets.update_counter(@table, key, {2, amount}, {key, 0, :infinity})
  end
end
```

---

## Step 856: Registry Advanced Usage

```elixir
defmodule MyApp.GameRegistry do
  # Named process registry with metadata

  def start_link do
    Registry.start_link(keys: :unique, name: __MODULE__)
  end

  def register_game(game_id, metadata) do
    Registry.register(__MODULE__, game_id, metadata)
  end

  def find_game(game_id) do
    case Registry.lookup(__MODULE__, game_id) do
      [{pid, metadata}] -> {:ok, pid, metadata}
      []                -> {:error, :not_found}
    end
  end

  def list_games do
    Registry.select(__MODULE__, [{{:_, :"$1", :"$2"}, [], [{{:"$1", :"$2"}}]}])
  end

  # Update metadata without reregistering
  def update_metadata(game_id, fun) do
    Registry.update_value(__MODULE__, game_id, fun)
  end

  # Dispatch to all processes with a specific metadata value
  def dispatch_to_player(player_id, message) do
    Registry.dispatch(__MODULE__, :player,
      fn entries ->
        for {pid, %{player_id: ^player_id}} <- entries do
          send(pid, message)
        end
      end
    )
  end
end

# Duplicate registry: multiple processes per key
Registry.start_link(keys: :duplicate, name: MyApp.TopicRegistry)

# Subscribe: multiple processes can register under same key
Registry.register(MyApp.TopicRegistry, "news", %{})

# Broadcast to all subscribers
Registry.dispatch(MyApp.TopicRegistry, "news", fn entries ->
  for {pid, _meta} <- entries, do: send(pid, {:news, "Breaking!..."})
end)
```

---

## Step 857: Task Supervision

```elixir
defmodule MyApp.TaskRunner do
  use Supervisor

  def start_link(_), do: Supervisor.start_link(__MODULE__, [], name: __MODULE__)

  def init(_) do
    children = [
      {Task.Supervisor, name: MyApp.TaskSupervisor}
    ]
    Supervisor.init(children, strategy: :one_for_one)
  end

  def run_async(fun) do
    Task.Supervisor.async(MyApp.TaskSupervisor, fun)
  end

  def run_async_nolink(fun) do
    Task.Supervisor.async_nolink(MyApp.TaskSupervisor, fun)
  end

  def run_many(funs) do
    funs
    |> Enum.map(&Task.Supervisor.async(MyApp.TaskSupervisor, &1))
    |> Task.await_many(30_000)
  end

  # Run with timeout and fallback
  def run_with_fallback(fun, fallback, timeout \\ 5_000) do
    task = Task.Supervisor.async_nolink(MyApp.TaskSupervisor, fun)

    case Task.yield(task, timeout) do
      {:ok, result} -> result
      nil ->
        Task.shutdown(task, :brutal_kill)
        fallback.()
    end
  end
end
```

---

## Step 858: Persistent Term Storage

```elixir
defmodule MyApp.Config do
  # :persistent_term: for rarely-changing global config
  # Faster than ETS for reads, but writes copy to all schedulers

  def load_config do
    config = %{
      db_pool_size:    10,
      cache_ttl:       3600,
      max_connections: 100,
      feature_flags:   %{new_ui: true, beta_api: false}
    }
    :persistent_term.put({__MODULE__, :config}, config)
  end

  def get(key) do
    config = :persistent_term.get({__MODULE__, :config})
    Map.get(config, key)
  end

  def update(key, value) do
    config = :persistent_term.get({__MODULE__, :config})
    :persistent_term.put({__MODULE__, :config}, Map.put(config, key, value))
  end

  # List all persistent terms
  def list do
    :persistent_term.get()
    |> Enum.filter(fn {{mod, _}, _} -> mod == __MODULE__ end)
  end

  # Erase when no longer needed
  def erase(key) do
    :persistent_term.erase({__MODULE__, key})
  end
end

# Compile-time constants via :persistent_term
defmodule MyApp.Constants do
  @on_load :init

  def init do
    :persistent_term.put(:app_version, "1.0.0")
    :persistent_term.put(:app_name,    "MyApp")
    :ok
  end

  def version, do: :persistent_term.get(:app_version)
  def name,    do: :persistent_term.get(:app_name)
end
```

---

## Step 859: GenStateMachine

```elixir
# mix.exs: {:gen_state_machine, "~> 3.0"}

defmodule MyApp.Connection do
  use GenStateMachine, callback_mode: :state_functions

  # States: disconnected, connecting, connected, reconnecting

  def start_link(config) do
    GenStateMachine.start_link(__MODULE__, config)
  end

  def connect(pid),    do: GenStateMachine.cast(pid, :connect)
  def disconnect(pid), do: GenStateMachine.cast(pid, :disconnect)
  def send_data(pid, data), do: GenStateMachine.call(pid, {:send, data})

  def init(config) do
    {:ok, :disconnected, %{config: config, socket: nil, retries: 0}}
  end

  # State: disconnected
  def disconnected(:cast, :connect, data) do
    case do_connect(data.config) do
      {:ok, socket} ->
        {:next_state, :connected, %{data | socket: socket, retries: 0}}
      {:error, _} ->
        {:next_state, :connecting, data, [{:state_timeout, 1000, :retry}]}
    end
  end

  def disconnected(event_type, event, data) do
    handle_common(event_type, event, :disconnected, data)
  end

  # State: connected
  def connected({:call, from}, {:send, payload}, %{socket: socket} = data) do
    case :gen_tcp.send(socket, payload) do
      :ok             -> {:keep_state, data, [{:reply, from, :ok}]}
      {:error, reason} ->
        {:next_state, :reconnecting, %{data | socket: nil},
         [{:reply, from, {:error, reason}}, {:state_timeout, 2000, :retry}]}
    end
  end

  def connected(:cast, :disconnect, data) do
    :gen_tcp.close(data.socket)
    {:next_state, :disconnected, %{data | socket: nil}}
  end

  def connected(:info, {:tcp_closed, _}, data) do
    {:next_state, :reconnecting, %{data | socket: nil},
     [{:state_timeout, 2000, :retry}]}
  end

  # State: reconnecting
  def reconnecting(:state_timeout, :retry, data) do
    case do_connect(data.config) do
      {:ok, socket} ->
        {:next_state, :connected, %{data | socket: socket, retries: 0}}
      {:error, _} ->
        new_data = %{data | retries: data.retries + 1}
        backoff  = min(30_000, 1000 * :math.pow(2, new_data.retries) |> round())
        {:keep_state, new_data, [{:state_timeout, backoff, :retry}]}
    end
  end

  defp handle_common(_, _, _, data), do: {:keep_state, data}
  defp do_connect(config), do: :gen_tcp.connect(config.host, config.port, [:binary], 5000)
end
```

---

## Step 860: Process Groups (pg)

```elixir
defmodule MyApp.ProcessGroups do
  # Erlang's :pg module for process group management
  # Better than global for ephemeral group membership

  def start do
    :pg.start_link()
  end

  def join(group) do
    :pg.join(__MODULE__, group, self())
  end

  def leave(group) do
    :pg.leave(__MODULE__, group, self())
  end

  def members(group) do
    :pg.get_members(__MODULE__, group)
  end

  def local_members(group) do
    :pg.get_local_members(__MODULE__, group)
  end

  def broadcast(group, message) do
    Enum.each(members(group), &send(&1, message))
  end

  def broadcast_local(group, message) do
    Enum.each(local_members(group), &send(&1, message))
  end

  # Example: shard a group for load balancing
  def get_member_for_key(group, key) do
    members = members(group)
    index   = rem(:erlang.phash2(key), length(members))
    Enum.at(members, index)
  end
end

# Usage in a LiveView
defmodule MyAppWeb.RoomLive do
  use MyAppWeb, :live_view

  def mount(%{"room_id" => room_id}, _session, socket) do
    if connected?(socket) do
      MyApp.ProcessGroups.join("room:#{room_id}")
    end
    {:ok, assign(socket, room_id: room_id)}
  end

  def handle_info({:room_message, msg}, socket) do
    {:noreply, push_event(socket, "message", msg)}
  end

  def terminate(_reason, socket) do
    MyApp.ProcessGroups.leave("room:#{socket.assigns.room_id}")
  end
end
```

---

## สรุป Part 79

✅ **Step 851** - GenServer state machine  
✅ **Step 852** - Process hibernation  
✅ **Step 853** - Dynamic supervisor  
✅ **Step 854** - Process monitoring  
✅ **Step 855** - ETS advanced patterns  
✅ **Step 856** - Registry advanced usage  
✅ **Step 857** - Task supervision  
✅ **Step 858** - Persistent term  
✅ **Step 859** - GenStateMachine  
✅ **Step 860** - Process groups  

➡️ [Part 80: Domain-Driven Design](./part-80-ddd.md)
